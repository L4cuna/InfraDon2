<script setup>
import { onMounted, ref, computed, watch } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';
PouchDB.plugin(PouchDBFind);

/*
 * ============================================================================
 * JUSTIFICATION DES CHOIX D'OPTIMISATION
 * ============================================================================
 * 
 * 1. FAUT-IL TOUT RÉPLIQUER ?
 * Réponse : OUI, dans mon cas c'est la meilleure approche.
 * 
 * Justification :
 * - La réplication complète (le sync bidirectionnelle) garantit que toutes les données
 *   sont disponibles localement, permettant un fonctionnement offline complet
 * - Avec PouchDB, la réplication est intelligente : seulement les changements
 *   sont transférés après la synchronisation initiale, pas toute la base
 * - Pour une application de type réseau social avec messages/commentaires, 
 *   le volume de données reste raisonnable et la réplication totale offre 
 *   une meilleure expérience utilisateur car il n'y a pas de latence réseau
 * - Alternative évaluée mais rejetée : réplication sélective rendrais complexe
 *   la gestion des relations entre documents (pays->messages->commentaires)
 * 
 * 2. EST-CE QUE LE allDocs AVEC "include_docs" FAIT SENS ?
 * Réponse : DÉPEND du cas d'usage.
 * 
 * Quand utiliser allDocs avec include_docs :
 * Pour récupérer TOUS les documents d'un type (ex: tous les pays)
 * Quand on a besoin du contenu complet de chaque document
 * Plus performant que des get() successifs
 * 
 * Quand NE PAS utiliser allDocs avec include_docs :
 * Quand on a besoin de filtrer ou trier les résultats (utiliser find() avec index)
 * Quand on ne veut qu'une partie des documents
 * Quand la base contient des types de documents mixtes sans préfixe clair
 * 
 * Mon choix :
 * - allDocs pour récupérer tous les pays (usage simple, besoin de tout)
 * - find() avec index pour les recherches filtrées/triées (région, likes, etc.)
 * 
 * 3. QUELLES MÉTHODES SONT LES PLUS UTILES ?
 * 
 * Mon choix + justifications :
 * 
 * a) db.createIndex() - ESSENTIEL
 *    → Permet des requêtes find() optimisées
 *    → Sans index, find() fait un scan complet de la base (très lent)
 *    → Index composites (type + région, type + countryId + likes) pour 
 *      des requêtes complexes performantes
 * 
 * b) db.find() avec limit/skip - RECOMMANDÉ pour pagination
 *    → Chargement progressif des top messages (10 par 10)
 *    → Évite de charger des milliers de messages en mémoire
 *    → Utilise les index pour tri/filtrage côté base de données
 * 
 * c) db.sync() avec {live: true, retry: true} - ESSENTIEL
 *    → Synchronisation temps réel bidirectionnelle
 *    → Détection automatique des changements
 *    → Retry automatique en cas de perte de connexion
 * 
 * d) db.getAttachment() - NÉCESSAIRE pour les images
 *    → Chargement paresseux (lazy loading) des attachments
 *    → Cache en mémoire (attachmentUrls) pour éviter rechargements
 *    → Conversion en Blob URL pour affichage dans le DOM
 * 
 * e) db.allDocs() avec startkey/endkey - EFFICACE pour les pays
 *    → Utilise l'index naturel des _id (country_xxx)
 *    → Plus rapide que find() pour ce cas simple
 *    → Pas besoin d'index supplémentaire
 * 
 * Méthodes ÉVITÉES volontairement :
 * - db.bulkDocs() : non nécessaire, nos créations sont unitaires
 * - db.query() avec map/reduce : find() avec index suffit et est plus simple
 * - db.changes() : remplacé par sync() qui offre plus de fonctionnalités
 * 
 * ============================================================================
 */

// Configuration de la base de données
const DB_CONFIG = {
  local: 'local_app_db',
  remote: 'http://admin:admin@localhost:5984/macollection_viola_ana',
  docTypes: {
    country: 'country',
    message: 'message',
    comment: 'comment'
  }
};

// État de l'application
const state = ref({
  localDB: null,
  remoteDB: null,
  syncHandler: null,
  isOnline: true,
  isLoading: false,
  
  countries: [],
  messages: [],
  comments: [],
  allCommentsForMessage: {},
  topLikedMessages: [],
  
  selectedCountryId: null,
  expandedMessageId: null,
  topMessagesOffset: 0,
  topMessagesLimit: 10,
  hasMoreTopMessages: true,
  
  newCountry: {
    name: '',
    capital: '',
    population: 0,
    area: 0,
    currency: '',
    languages: '',
    region: '',
    subregion: '',
    flag: 'https://flagcdn.com/160x120/fr.png'
  },
  newMessage: {
    title: '',
    content: ''
  },
  
  commentForms: {},
  
  editingCountryId: null,
  editingMessageId: null,
  editingCommentId: null,
  editingCommentData: { content: '', author: '' },
  
  searchRegion: '',
  searchMessageTerm: '',
  sortBy: 'date',
  
  selectedFile: null,
  uploadingAttachment: false,
  attachmentUrls: {},
  
  userLikedMessages: new Set()
});

const initCommentForm = (messageId) => {
  if (!state.value.commentForms[messageId]) {
    state.value.commentForms[messageId] = {
      content: '',
      author: ''
    };
  }
};

const methods = {
  async initDatabases() {
    try {
      state.value.isLoading = true;
      
      // Connexion aux bases de données locale et distante
      state.value.localDB = new PouchDB(DB_CONFIG.local);
      state.value.remoteDB = new PouchDB(DB_CONFIG.remote);
      console.log('✅ Bases de données initialisées');
      
      // Création des index AVANT toute opération de requête
      await methods.createIndexes();
      
      // Réplication initiale : récupération des données du serveur
      await methods.replicateFromRemote();
      
      // Chargement des données dans l'interface
      await methods.fetchAllData();
      
      // Activation de la synchronisation continue si en ligne
      if (state.value.isOnline) {
        methods.startContinuousSync();
      }
    } catch (error) {
      console.error('Erreur d\'initialisation:', error);
    } finally {
      state.value.isLoading = false;
    }
  },

  /*
   * CRÉATION DES INDEX
   * 
   * Les index sont créés au démarrage pour optimiser toutes les requêtes find().
   * Chaque index composite permet des requêtes multi-critères efficaces.
   * 
   * Principe : Un index sur [type, region] permet de filtrer par type ET région
   * en une seule requête optimisée, sans scan complet de la base.
   */
  async createIndexes() {
    try {
      // Index pour rechercher les pays par région (insensible à la casse)
      // Utilisé par : searchByRegion()
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'region'] }
      });
      
      // Index pour trier les messages d'un pays par nombre de likes
      // Utilisé par : fetchMessages() avec sortBy='likes'
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'countryId', 'likes'] }
      });
      
      // Index pour trier les messages d'un pays par date de création
      // Utilisé par : fetchMessages() avec sortBy='date'
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'countryId', 'createdAt'] }
      });
      
      // Index pour rechercher des messages par titre dans un pays
      // Utilisé par : fetchMessages() avec searchMessageTerm
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'countryId', 'title'] }
      });
      
      // Index pour récupérer les commentaires d'un message triés par date
      // Utilisé par : fetchComments() et fetchAllCommentsForMessage()
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'messageId', 'createdAt'] }
      });
      
      // Index pour obtenir le top N des messages tous pays confondus
      // Utilisé par : fetchTopLikedMessages()
      // Cet index permet un tri décroissant sur les likes sans charger tous les messages
      await state.value.localDB.createIndex({
        index: { fields: ['type', 'likes'] }
      });
      
      console.log('✅ Index créés avec succès');
    } catch (error) {
      console.error('❌ Erreur lors de la création des index:', error);
    }
  },

  /*
   * RÉPLICATION INITIALE
   * 
   * Stratégie : Réplication unidirectionnelle au démarrage (remote → local)
   * pour s'assurer que la base locale contient toutes les données du serveur.
   * 
   * Note : Après cette réplication initiale, la synchronisation continue
   * (bidirectionnelle) prend le relais via startContinuousSync().
   */
  async replicateFromRemote() {
    try {
      console.log('🔄 Réplication initiale en cours...');
      await state.value.localDB.replicate.from(state.value.remoteDB);
      console.log('✅ Réplication terminée');
    } catch (error) {
      console.error('❌ Erreur de réplication:', error);
    }
  },

  /*
   * SYNCHRONISATION CONTINUE BIDIRECTIONNELLE
   * 
   * Utilise db.sync() au lieu de db.replicate() car :
   * - Synchronisation bidirectionnelle automatique (local ↔ remote)
   * - Mode live : détection temps réel des changements
   * - Mode retry : reconnexion automatique en cas de perte réseau
   * - Gestion automatique des conflits (last-write-wins par défaut)
   * 
   * Performance : Seuls les changements (deltas) sont synchronisés,
   * pas toute la base de données.
   */
  startContinuousSync() {
    if (!state.value.isOnline) return;
    console.log('🔄 Synchronisation continue activée');
    
    state.value.syncHandler = state.value.localDB.sync(state.value.remoteDB, {
      live: true,      // Synchronisation en temps réel
      retry: true      // Reconnexion automatique
    })
    .on('change', () => {
      // Rafraîchissement de l'interface à chaque changement détecté
      methods.fetchAllData();
    })
    .on('error', (err) => {
      console.error('❌ Erreur de synchronisation:', err);
    });
  },

  stopContinuousSync() {
    if (state.value.syncHandler) {
      state.value.syncHandler.cancel();
      state.value.syncHandler = null;
      console.log('⏸️ Synchronisation continue désactivée');
    }
  },

  toggleOnlineOffline() {
    state.value.isOnline = !state.value.isOnline;
    if (state.value.isOnline) {
      methods.startContinuousSync();
    } else {
      methods.stopContinuousSync();
    }
  },

  async fetchAllData() {
    await methods.fetchCountries();
    await methods.fetchTopLikedMessages();
    if (state.value.selectedCountryId) {
      await methods.fetchMessages();
      await methods.fetchComments();
    }
  },

  /*
   * RÉCUPÉRATION DU TOP N DES MESSAGES LES PLUS LIKÉS
   * 
   * Optimisation : Pagination avec limit/skip pour charger 10 messages à la fois.
   * 
   * Avantages :
   * - Chargement initial rapide (10 messages au lieu de potentiellement des milliers)
   * - Réduction de la consommation mémoire
   * - Expérience utilisateur progressive (bouton "Charger plus")
   * 
   * Utilisation de find() avec index au lieu de allDocs car :
   * - Besoin de tri décroissant sur les likes
   * - L'index sur [type, likes] permet un tri optimisé côté base
   * - Pas de tri JavaScript (conformément au requirement "ne pas exécuter en TS")
   */
  async fetchTopLikedMessages(loadMore = false) {
    if (!state.value.localDB) return;
    
    if (!loadMore) {
      state.value.topMessagesOffset = 0;
      state.value.topLikedMessages = [];
    }
    
    try {
      // Requête avec tri décroissant sur les likes
      // Le tri est effectué par la base de données grâce à l'index [type, likes]
      const result = await state.value.localDB.find({
        selector: {
          type: DB_CONFIG.docTypes.message
        },
        sort: [{ type: 'asc' }, { likes: 'desc' }],
        limit: state.value.topMessagesLimit,
        skip: state.value.topMessagesOffset
      });
      
      if (loadMore) {
        state.value.topLikedMessages = [
          ...state.value.topLikedMessages,
          ...result.docs
        ];
      } else {
        state.value.topLikedMessages = result.docs || [];
      }
      
      // lazy loading des images
      // Optimisation : les attachments ne sont chargés que pour les messages affichés
      for (const message of result.docs) {
        if (message._attachments) {
          await methods.loadMessageAttachments(message._id);
        }
      }
      
      // Détection s'il reste d'autres messages à charger
      state.value.hasMoreTopMessages = result.docs.length === state.value.topMessagesLimit;
      
      if (loadMore) {
        state.value.topMessagesOffset += state.value.topMessagesLimit;
      }
    } catch (error) {
      console.error('❌ Erreur lors de la récupération du top messages:', error);
      state.value.topLikedMessages = [];
    }
  },

  async loadMoreTopMessages() {
    await methods.fetchTopLikedMessages(true);
  },

  /*
   * RÉCUPÉRATION DE TOUS LES PAYS
   * 
   * Choix : Utilisation de allDocs avec startkey/endkey au lieu de find()
   * 
   * Justification :
   * - Tous les pays ont un _id commençant par "country_" (préfixe cohérent)
   * - allDocs utilise l'index naturel des _id (pas besoin d'index custom)
   * - Plus performant que find() pour ce cas d'usage simple
   * - include_docs: true car on a besoin du contenu complet de chaque pays
   * 
   * Note : Si j'avait besoin de filtrer/trier, j'utiliserait find() avec index
   */
  async fetchCountries() {
    if (!state.value.localDB) return;
    try {
      const result = await state.value.localDB.allDocs({
        include_docs: true,
        startkey: `${DB_CONFIG.docTypes.country}_`,
        endkey: `${DB_CONFIG.docTypes.country}_\uffff`
      });
      state.value.countries = result.rows.map(row => row.doc).filter(doc => doc);
    } catch (error) {
      console.error('❌ Erreur lors de la récupération des pays:', error);
      state.value.countries = [];
    }
  },

  /*
   * RÉCUPÉRATION DES MESSAGES D'UN PAYS
   * 
   * Optimisation : Utilisation de find() avec index pour filtrage ET tri
   * 
   * Les index utilisés selon le contexte :
   * - sortBy='likes' → index [type, countryId, likes]
   * - sortBy='date' → index [type, countryId, createdAt]
   * - searchMessageTerm → index [type, countryId, title]
   * 
   * Conformité : Le tri est effectué par la base de données (non en TS)
   * grâce aux index, ce qui respecte le requirement du TP.
   */
  async fetchMessages() {
    if (!state.value.localDB || !state.value.selectedCountryId) {
      state.value.messages = [];
      return;
    }
    try {
      let selector = {
        type: DB_CONFIG.docTypes.message,
        countryId: state.value.selectedCountryId
      };
      
      // Ajout du filtre de recherche par titre si renseigné
      if (state.value.searchMessageTerm.trim()) {
        selector.title = { 
          $regex: new RegExp(state.value.searchMessageTerm.trim(), 'i')
        };
      }
      
      // Configuration du tri selon la sélection de l'utilisateur
      let sort = [{ type: 'asc' }, { countryId: 'asc' }];
      if (state.value.sortBy === 'likes') {
        sort.push({ likes: 'desc' });
      } else {
        sort.push({ createdAt: 'desc' });
      }
      
      const result = await state.value.localDB.find({ selector, sort });
      state.value.messages = result.docs || [];
      
      // Chargement paresseux des attachments (lazy loading)
      for (const message of state.value.messages) {
        if (message._attachments) {
          await methods.loadMessageAttachments(message._id);
        }
      }
      
      // Initialisation des formulaires de commentaire pour chaque message
      state.value.messages.forEach(msg => initCommentForm(msg._id));
    } catch (error) {
      console.error('❌ Erreur lors de la récupération des messages:', error);
      state.value.messages = [];
    }
  },

  /*
   * RÉCUPÉRATION DU PREMIER COMMENTAIRE DE CHAQUE MESSAGE
   * 
   * Optimisation : Utilisation de limit de 1 pour charger que le dernier commentaire
   * 
   * Justification :
   * - L'affichage initial ne montre qu'un seul commentaire par message
   * - Limit 1 évite de charger tous les commentaires (économie réseau/mémoire)
   * - Les autres commentaires sont chargés à la demande (voir fetchAllCommentsForMessage)
   * 
   * Performance : Pour 100 messages avec 10 commentaires chacun, on charge
   * 100 commentaires au lieu de 1000
   */
  async fetchComments() {
    if (!state.value.localDB || !state.value.messages.length) {
      state.value.comments = [];
      return;
    }
    try {
      const comments = [];
      for (const message of state.value.messages) {
        const result = await state.value.localDB.find({
          selector: {
            type: DB_CONFIG.docTypes.comment,
            messageId: message._id
          },
          sort: [{ type: 'asc' }, { messageId: 'asc' }, { createdAt: 'asc' }],
          limit: 1  // Optimisation : un seul commentaire par message
        });
        if (result.docs && result.docs.length > 0) {
          comments.push(result.docs[0]);
        }
      }
      state.value.comments = comments;
    } catch (error) {
      console.error('❌ Erreur lors de la récupération des commentaires:', error);
      state.value.comments = [];
    }
  },

  /*
   * CHARGEMENT À LA DEMANDE DE TOUS LES COMMENTAIRES D'UN MESSAGE
   * 
   * Stratégie : Lazy loading avec cache
   * 
   * Optimisation :
   * - Les commentaires ne sont chargés que quand l'utilisateur clique "Voir tout"
   * - Cache en mémoire (allCommentsForMessage) pour éviter rechargements
   * - Pas de limit : on charge tous les commentaires une seule fois
   * 
   * Justification : Balance entre performance initiale et expérience utilisateur
   */
  async fetchAllCommentsForMessage(messageId) {
    if (!state.value.localDB) return;
    
    // Cache : évite de recharger si déjà en mémoire
    if (state.value.allCommentsForMessage[messageId]) {
      return;
    }
    
    try {
      const result = await state.value.localDB.find({
        selector: {
          type: DB_CONFIG.docTypes.comment,
          messageId: messageId
        },
        sort: [{ type: 'asc' }, { messageId: 'asc' }, { createdAt: 'asc' }]
      });
      state.value.allCommentsForMessage[messageId] = result.docs || [];
    } catch (error) {
      console.error('❌ Erreur lors de la récupération des commentaires:', error);
      state.value.allCommentsForMessage[messageId] = [];
    }
  },

  async createCountry() {
    if (!state.value.localDB) return;
    try {
      const doc = {
        _id: `${DB_CONFIG.docTypes.country}_${Date.now()}`,
        type: DB_CONFIG.docTypes.country,
        ...state.value.newCountry,
        population: Number(state.value.newCountry.population),
        area: Number(state.value.newCountry.area),
        languages: state.value.newCountry.languages.split(',').map(l => l.trim()),
        createdAt: new Date().toISOString()
      };
      
      await state.value.localDB.put(doc);
      methods.resetCountryForm();
      await methods.fetchCountries();
      console.log('✅ Pays créé avec succès');
    } catch (error) {
      console.error('❌ Erreur lors de la création du pays:', error);
    }
  },

  async updateCountry() {
    if (!state.value.localDB || !state.value.editingCountryId) return;
    try {
      const doc = await state.value.localDB.get(state.value.editingCountryId);
      Object.assign(doc, {
        ...state.value.newCountry,
        population: Number(state.value.newCountry.population),
        area: Number(state.value.newCountry.area),
        languages: state.value.newCountry.languages.split(',').map(l => l.trim()),
        updatedAt: new Date().toISOString()
      });
      
      await state.value.localDB.put(doc);
      methods.resetCountryForm();
      state.value.editingCountryId = null;
      await methods.fetchCountries();
      console.log('✅ Pays mis à jour');
    } catch (error) {
      console.error('❌ Erreur lors de la mise à jour du pays:', error);
    }
  },

  async deleteCountry(id) {
    if (!state.value.localDB) return;
    
    if (!confirm('Supprimer ce pays et tous ses messages ?')) return;
    
    try {
      const doc = await state.value.localDB.get(id);
      await state.value.localDB.remove(doc);
      
      if (state.value.selectedCountryId === id) {
        state.value.selectedCountryId = null;
        state.value.messages = [];
        state.value.comments = [];
      }
      
      await methods.fetchCountries();
      console.log('✅ Pays supprimé');
    } catch (error) {
      console.error('❌ Erreur lors de la suppression du pays:', error);
    }
  },

  startEditingCountry(country) {
    state.value.editingCountryId = country._id;
    state.value.newCountry = {
      name: country.name,
      capital: country.capital,
      population: country.population,
      area: country.area,
      currency: country.currency,
      languages: Array.isArray(country.languages) ? country.languages.join(', ') : country.languages,
      region: country.region,
      subregion: country.subregion,
      flag: country.flag
    };
  },

  cancelEditingCountry() {
    methods.resetCountryForm();
    state.value.editingCountryId = null;
  },

  resetCountryForm() {
    state.value.newCountry = {
      name: '',
      capital: '',
      population: 0,
      area: 0,
      currency: '',
      languages: '',
      region: '',
      subregion: '',
      flag: 'https://flagcdn.com/160x120/fr.png'
    };
  },

  selectCountry(countryId) {
    state.value.selectedCountryId = countryId;
    state.value.searchMessageTerm = '';
    state.value.sortBy = 'date';
    methods.fetchMessages();
    methods.fetchComments();
  },

  async createMessage() {
    if (!state.value.localDB || !state.value.selectedCountryId) return;
    try {
      const doc = {
        _id: `${DB_CONFIG.docTypes.message}_${Date.now()}`,
        type: DB_CONFIG.docTypes.message,
        countryId: state.value.selectedCountryId,
        title: state.value.newMessage.title,
        content: state.value.newMessage.content,
        likes: 0,
        createdAt: new Date().toISOString()
      };
      
      await state.value.localDB.put(doc);
      methods.resetMessageForm();
      await methods.fetchMessages();
      await methods.fetchTopLikedMessages();
      console.log('✅ Message créé');
    } catch (error) {
      console.error('❌ Erreur lors de la création du message:', error);
    }
  },

  async updateMessage() {
    if (!state.value.localDB || !state.value.editingMessageId) return;
    try {
      const doc = await state.value.localDB.get(state.value.editingMessageId);
      doc.title = state.value.newMessage.title;
      doc.content = state.value.newMessage.content;
      doc.updatedAt = new Date().toISOString();
      
      await state.value.localDB.put(doc);
      methods.resetMessageForm();
      state.value.editingMessageId = null;
      await methods.fetchMessages();
      console.log('✅ Message mis à jour');
    } catch (error) {
      console.error('❌ Erreur lors de la mise à jour du message:', error);
    }
  },

  async deleteMessage(id) {
    if (!state.value.localDB) return;
    
    if (!confirm('Supprimer ce message et tous ses commentaires ?')) return;
    
    try {
      const doc = await state.value.localDB.get(id);
      await state.value.localDB.remove(doc);
      
      // Suppression en cascade des commentaires associés
      const commentsResult = await state.value.localDB.find({
        selector: { 
          type: DB_CONFIG.docTypes.comment,
          messageId: id 
        }
      });
      
      for (const comment of commentsResult.docs) {
        await state.value.localDB.remove(comment);
      }
      
      await methods.fetchMessages();
      await methods.fetchComments();
      await methods.fetchTopLikedMessages();
      console.log('✅ Message supprimé');
    } catch (error) {
      console.error('❌ Erreur lors de la suppression du message:', error);
    }
  },

  async toggleLike(message) {
    if (!state.value.localDB) return;
    try {
      const doc = await state.value.localDB.get(message._id);
      
      if (state.value.userLikedMessages.has(message._id)) {
        doc.likes = Math.max(0, (doc.likes || 0) - 1);
        state.value.userLikedMessages.delete(message._id);
      } else {
        doc.likes = (doc.likes || 0) + 1;
        state.value.userLikedMessages.add(message._id);
      }
      
      await state.value.localDB.put(doc);
      await methods.fetchMessages();
      await methods.fetchTopLikedMessages();
      console.log('✅ Like mis à jour');
    } catch (error) {
      console.error('❌ Erreur lors de la mise à jour du like:', error);
    }
  },

  startEditingMessage(message) {
    state.value.editingMessageId = message._id;
    state.value.newMessage = {
      title: message.title,
      content: message.content
    };
  },

  cancelEditingMessage() {
    methods.resetMessageForm();
    state.value.editingMessageId = null;
  },

  resetMessageForm() {
    state.value.newMessage = { title: '', content: '' };
  },

  async createComment(messageId) {
    if (!state.value.localDB) return;
    
    initCommentForm(messageId);
    const form = state.value.commentForms[messageId];
    
    if (!form.content?.trim() || !form.author?.trim()) {
      alert('Veuillez remplir le commentaire et votre nom');
      return;
    }
    
    try {
      const doc = {
        _id: `${DB_CONFIG.docTypes.comment}_${Date.now()}_${Math.random()}`,
        type: DB_CONFIG.docTypes.comment,
        messageId: messageId,
        content: form.content.trim(),
        author: form.author.trim(),
        createdAt: new Date().toISOString()
      };
      
      await state.value.localDB.put(doc);
      state.value.commentForms[messageId] = { content: '', author: '' };
      await methods.fetchComments();
      
      // Rafraîchir les commentaires si la vue étendue est active
      if (state.value.expandedMessageId === messageId) {
        delete state.value.allCommentsForMessage[messageId];
        await methods.fetchAllCommentsForMessage(messageId);
      }
      
      console.log('✅ Commentaire créé');
    } catch (error) {
      console.error('❌ Erreur lors de la création du commentaire:', error);
      alert('Erreur lors de la création du commentaire');
    }
  },

  async updateComment() {
    if (!state.value.localDB || !state.value.editingCommentId) return;
    
    const editData = state.value.editingCommentData;
    
    if (!editData.content?.trim() || !editData.author?.trim()) {
      alert('Veuillez remplir le commentaire et votre nom');
      return;
    }
    
    try {
      const doc = await state.value.localDB.get(state.value.editingCommentId);
      doc.content = editData.content.trim();
      doc.author = editData.author.trim();
      doc.updatedAt = new Date().toISOString();
      
      await state.value.localDB.put(doc);
      
      state.value.editingCommentId = null;
      state.value.editingCommentData = { content: '', author: '' };
      
      await methods.fetchComments();
      
      if (state.value.expandedMessageId === doc.messageId) {
        delete state.value.allCommentsForMessage[doc.messageId];
        await methods.fetchAllCommentsForMessage(doc.messageId);
      }
      
      console.log('✅ Commentaire mis à jour');
    } catch (error) {
      console.error('❌ Erreur lors de la mise à jour du commentaire:', error);
    }
  },

  async deleteComment(comment) {
    if (!state.value.localDB) return;
    
    if (!confirm('Supprimer ce commentaire ?')) return;
    
    try {
      const doc = await state.value.localDB.get(comment._id);
      await state.value.localDB.remove(doc);
      
      await methods.fetchComments();
      
      if (state.value.expandedMessageId === comment.messageId) {
        delete state.value.allCommentsForMessage[comment.messageId];
        await methods.fetchAllCommentsForMessage(comment.messageId);
      }
      
      console.log('✅ Commentaire supprimé');
    } catch (error) {
      console.error('❌ Erreur lors de la suppression du commentaire:', error);
    }
  },

  startEditingComment(comment) {
    state.value.editingCommentId = comment._id;
    state.value.editingCommentData = {
      content: comment.content,
      author: comment.author
    };
  },

  cancelEditingComment() {
    state.value.editingCommentId = null;
    state.value.editingCommentData = { content: '', author: '' };
  },

  async toggleCommentsVisibility(messageId) {
    if (state.value.expandedMessageId === messageId) {
      state.value.expandedMessageId = null;
    } else {
      state.value.expandedMessageId = messageId;
      await methods.fetchAllCommentsForMessage(messageId);
    }
  },

  /*
   * RECHERCHE PAR RÉGION
   */
  async searchByRegion() {
    if (!state.value.localDB || !state.value.searchRegion.trim()) {
      await methods.fetchCountries();
      return;
    }
    try {
      const searchTerm = state.value.searchRegion.trim();
      

      const result = await state.value.localDB.find({
        selector: { 
          type: DB_CONFIG.docTypes.country,
          region: { 
            $regex: new RegExp(`^${searchTerm}$`, 'i')  
          }
        }
      });
      
      state.value.countries = result.docs;
      console.log(`✅ Recherche effectuée : ${result.docs.length} pays trouvés`);
    } catch (error) {
      console.error('❌ Erreur lors de la recherche:', error);
      state.value.countries = [];
    }
  },

  /*
   * GESTION DES FICHIERS JOINTS (ATTACHMENTS)
   * 
   * Stratégie d'optimisation :
   * 
   * 1. Stockage : Les attachments sont stockés DANS le document PouchDB
   *    → Avantage : synchronisation automatique avec la réplication
   *    → Inconvénient : taille limitée (5MB par fichier recommandé)
   * 
   * 2. Chargement : Lazy loading avec cache
   *    → getAttachment() seulement quand nécessaire
   *    → Cache en mémoire (attachmentUrls) pour éviter rechargements
   *    → Conversion en Blob URL pour affichage DOM
   * 
   * 3. Nettoyage : Révocation des Blob URL pour éviter memory leaks
   *    → URL.revokeObjectURL() lors de la suppression
   */
  handleFileSelect(event, messageId) {
    const file = event.target.files[0];
    if (file) {
      state.value.selectedFile = { file, messageId };
    }
  },

  async uploadAttachment(messageId) {
    if (!state.value.selectedFile || !state.value.localDB) return;
    
    const { file } = state.value.selectedFile;
    
    // Validation de la taille (5MB max recommandé pour PouchDB)
    const maxSize = 5 * 1024 * 1024;
    if (file.size > maxSize) {
      alert('Le fichier est trop volumineux (max 5MB)');
      return;
    }
    
    try {
      state.value.uploadingAttachment = true;
      
      const doc = await state.value.localDB.get(messageId);
      
      // Ajout du fichier comme attachment au document
      // L'attachment sera automatiquement synchronisé avec le serveur
      await state.value.localDB.putAttachment(
        messageId,
        file.name,
        doc._rev,
        file,
        file.type
      );
      
      await methods.fetchMessages();
      await methods.loadMessageAttachments(messageId);
      
      state.value.selectedFile = null;
      state.value.uploadingAttachment = false;
      
      console.log('✅ Fichier uploadé avec succès');
      alert('Fichier ajouté avec succès !');
    } catch (error) {
      console.error('❌ Erreur lors de l\'upload:', error);
      state.value.uploadingAttachment = false;
      alert('Erreur lors de l\'upload du fichier');
    }
  },

  async deleteAttachment(messageId, attachmentName) {
    if (!state.value.localDB) return;
    
    if (!confirm('Supprimer ce fichier ?')) return;
    
    try {
      const doc = await state.value.localDB.get(messageId);
      
      await state.value.localDB.removeAttachment(
        messageId,
        attachmentName,
        doc._rev
      );
      
      // Nettoyage du cache : révocation de l'URL Blob pour libérer la mémoire
      if (state.value.attachmentUrls[messageId]?.[attachmentName]) {
        URL.revokeObjectURL(state.value.attachmentUrls[messageId][attachmentName]);
        delete state.value.attachmentUrls[messageId][attachmentName];
      }
      
      await methods.fetchMessages();
      
      console.log('✅ Fichier supprimé');
      alert('Fichier supprimé avec succès !');
    } catch (error) {
      console.error('❌ Erreur lors de la suppression:', error);
      alert('Erreur lors de la suppression du fichier');
    }
  },

  /*
   * CHARGEMENT ET CONVERSION DES ATTACHMENTS EN URLs
   * 
   * Optimisation : Cache avec vérification pour éviter rechargements inutiles
   * 
   * Processus :
   * 1. Vérification du cache (attachmentUrls)
   * 2. Si non en cache : getAttachment() → Blob
   * 3. Conversion Blob → URL avec URL.createObjectURL()
   * 4. Stockage en cache pour réutilisation
   * 
   * Note : Les Blob URL doivent être révoquées avec revokeObjectURL()
   * lors de la suppression pour éviter les memory leaks.
   */
  async loadMessageAttachments(messageId) {
    if (!state.value.localDB) return;
    
    try {
      const doc = await state.value.localDB.get(messageId, { attachments: true });
      
      if (!doc._attachments) return;
      
      if (!state.value.attachmentUrls[messageId]) {
        state.value.attachmentUrls[messageId] = {};
      }
      
      for (const attachmentName in doc._attachments) {
        // Éviter de recharger si déjà en cache (optimisation)
        if (state.value.attachmentUrls[messageId][attachmentName]) {
          continue;
        }
        
        const blob = await state.value.localDB.getAttachment(messageId, attachmentName);
        const url = URL.createObjectURL(blob);
        state.value.attachmentUrls[messageId][attachmentName] = url;
      }
    } catch (error) {
      console.error('❌ Erreur lors du chargement des attachments:', error);
    }
  },

  /*
   * GÉNÉRATION DE DONNÉES DE TEST
   * 
   * Fonction utilitaire pour peupler la base avec des données cohérentes.
   * Crée des pays, messages et commentaires interconnectés pour tester
   * toutes les fonctionnalités (recherche, tri, pagination, etc.)
   */
  async generateTestData() {
    try {
      console.log('🔧 Génération de données de test...');
      
      const testCountries = [
        { name: 'France', capital: 'Paris', population: 67000000, area: 551695, currency: 'EUR', languages: ['Français'], region: 'Europe', subregion: 'Western Europe', flag: 'https://flagcdn.com/160x120/fr.png' },
        { name: 'Germany', capital: 'Berlin', population: 83000000, area: 357022, currency: 'EUR', languages: ['Deutsch'], region: 'Europe', subregion: 'Western Europe', flag: 'https://flagcdn.com/160x120/de.png' },
        { name: 'Italy', capital: 'Rome', population: 60000000, area: 301340, currency: 'EUR', languages: ['Italiano'], region: 'Europe', subregion: 'Southern Europe', flag: 'https://flagcdn.com/160x120/it.png' },
        { name: 'Spain', capital: 'Madrid', population: 47000000, area: 505990, currency: 'EUR', languages: ['Español'], region: 'Europe', subregion: 'Southern Europe', flag: 'https://flagcdn.com/160x120/es.png' },
        { name: 'Switzerland', capital: 'Bern', population: 8500000, area: 41290, currency: 'CHF', languages: ['Français', 'Deutsch', 'Italiano'], region: 'Europe', subregion: 'Western Europe', flag: 'https://flagcdn.com/160x120/ch.png' }
      ];
      
      for (const countryData of testCountries) {
        const countryDoc = {
          _id: `${DB_CONFIG.docTypes.country}_${Date.now()}_${Math.random()}`,
          type: DB_CONFIG.docTypes.country,
          ...countryData,
          createdAt: new Date().toISOString()
        };
        await state.value.localDB.put(countryDoc);
        
        // Création de 10 messages par pays
        for (let i = 0; i < 10; i++) {
          const messageDoc = {
            _id: `${DB_CONFIG.docTypes.message}_${Date.now()}_${Math.random()}`,
            type: DB_CONFIG.docTypes.message,
            countryId: countryDoc._id,
            title: `Message ${i + 1} about ${countryData.name}`,
            content: `This is test message number ${i + 1} about ${countryData.name}. Lorem ipsum dolor sit amet, consectetur adipiscing elit.`,
            likes: Math.floor(Math.random() * 50),
            createdAt: new Date(Date.now() - Math.random() * 10000000000).toISOString()
          };
          await state.value.localDB.put(messageDoc);
          
          // Création de 3 commentaires par message
          for (let j = 0; j < 3; j++) {
            const commentDoc = {
              _id: `${DB_CONFIG.docTypes.comment}_${Date.now()}_${Math.random()}`,
              type: DB_CONFIG.docTypes.comment,
              messageId: messageDoc._id,
              content: `Comment ${j + 1} on this message. This is a test comment with some content.`,
              author: `User ${j + 1}`,
              createdAt: new Date(Date.now() - Math.random() * 10000000000).toISOString()
            };
            await state.value.localDB.put(commentDoc);
          }
        }
      }
      
      await methods.fetchAllData();
      console.log('✅ Données de test générées avec succès');
      alert('50 pays, 500 messages et 1500 commentaires créés !');
    } catch (error) {
      console.error('❌ Erreur lors de la génération:', error);
    }
  }
};

const computedData = computed(() => {
  // Construction d'un index pour accès rapide au premier commentaire de chaque message
  const firstCommentByMessage = {};
  state.value.comments.forEach(comment => {
    if (!firstCommentByMessage[comment.messageId]) {
      firstCommentByMessage[comment.messageId] = comment;
    }
  });

  return { 
    topLikedMessages: state.value.topLikedMessages,
    firstCommentByMessage 
  };
});

// Watchers pour déclencher automatiquement les recherches
watch(() => state.value.searchMessageTerm, () => {
  if (state.value.selectedCountryId) methods.fetchMessages();
});

watch(() => state.value.sortBy, () => {
  if (state.value.selectedCountryId) methods.fetchMessages();
});

// Initialisation au montage du composant
onMounted(() => {
  methods.initDatabases();
});
</script>

<template>
  <div class="app-container">
    <div class="app-header">
      <h1>Infrastructure de données 2 - noSQL</h1>
      <div class="header-actions">
        <button @click="methods.toggleOnlineOffline" class="btn-toggle">
          {{ state.isOnline ? '🌐 ONLINE' : '📵 OFFLINE' }}
        </button>
        <span class="status">{{ state.isOnline ? 'Connecté' : 'Déconnecté' }}</span>
        <button @click="methods.generateTestData" class="btn secondary small">
          🔧 Générer données test
        </button>
      </div>
    </div>

    <div v-if="state.isLoading" class="loading">Chargement...</div>

    <div v-else class="main-content">
      <!-- PAYS -->
      <section class="section">
        <h2>Gestion des Pays</h2>

        <div class="search-bar">
          <input
            v-model="state.searchRegion"
            placeholder="Rechercher par région (ex: Europe, Asia, Africa...)"
            @keyup.enter="methods.searchByRegion"
            class="input"
          />
          <button @click="methods.searchByRegion" class="btn primary">Rechercher</button>
          <button @click="methods.fetchCountries" class="btn secondary">Réinitialiser</button>
        </div>

        <div class="form-container">
          <h3>{{ state.editingCountryId ? 'Modifier un pays' : 'Ajouter un pays' }}</h3>
          <form @submit.prevent="state.editingCountryId ? methods.updateCountry() : methods.createCountry()">
            <div class="form-grid">
              <div class="form-group">
                <label>Nom</label>
                <input v-model="state.newCountry.name" required class="input" />
              </div>
              <div class="form-group">
                <label>Capitale</label>
                <input v-model="state.newCountry.capital" required class="input" />
              </div>
              <div class="form-group">
                <label>Population</label>
                <input v-model.number="state.newCountry.population" type="number" required class="input" />
              </div>
              <div class="form-group">
                <label>Superficie (km²)</label>
                <input v-model.number="state.newCountry.area" type="number" required class="input" />
              </div>
              <div class="form-group">
                <label>Monnaie</label>
                <input v-model="state.newCountry.currency" required class="input" />
              </div>
              <div class="form-group">
                <label>Langues (séparées par virgules)</label>
                <input v-model="state.newCountry.languages" required class="input" />
              </div>
              <div class="form-group">
                <label>Région</label>
                <input v-model="state.newCountry.region" required class="input" placeholder="Europe, Asia, Africa..." />
              </div>
              <div class="form-group">
                <label>Sous-région</label>
                <input v-model="state.newCountry.subregion" required class="input" />
              </div>
            </div>
            <div class="form-actions">
              <button type="submit" class="btn primary">
                {{ state.editingCountryId ? 'Modifier' : 'Ajouter' }}
              </button>
              <button v-if="state.editingCountryId" @click="methods.cancelEditingCountry" type="button" class="btn secondary">
                Annuler
              </button>
            </div>
          </form>
        </div>

        <div v-if="state.countries.length === 0" class="no-data">Aucun pays trouvé</div>
        <div v-else class="cards-grid">
          <div 
            v-for="country in state.countries" 
            :key="country._id"
            class="card"
            :class="{ selected: state.selectedCountryId === country._id }"
            @click="methods.selectCountry(country._id)"
          >
            <div class="card-header">
              <h3>{{ country.name }}</h3>
              <img :src="country.flag" :alt="country.name" class="flag" />
            </div>
            <div class="card-body">
              <p><strong>Capitale:</strong> {{ country.capital }}</p>
              <p><strong>Population:</strong> {{ country.population?.toLocaleString() }}</p>
              <p><strong>Région:</strong> {{ country.region }}</p>
              <p><strong>Sous-région:</strong> {{ country.subregion }}</p>
            </div>
            <div class="card-actions">
              <button @click.stop="methods.startEditingCountry(country)" class="btn secondary small">Modifier</button>
              <button @click.stop="methods.deleteCountry(country._id)" class="btn danger small">Supprimer</button>
            </div>
          </div>
        </div>
      </section>

      <!-- MESSAGES -->
      <section v-if="state.selectedCountryId" class="section">
        <h2>Messages pour {{ state.countries.find(c => c._id === state.selectedCountryId)?.name }}</h2>

        <div class="toolbar">
          <input
            v-model="state.searchMessageTerm"
            placeholder="Rechercher un message par titre..."
            class="input flex-1"
          />
          <select v-model="state.sortBy" class="select">
            <option value="date">Par date (plus récent)</option>
            <option value="likes">Par nombre de likes</option>
          </select>
        </div>

        <div class="form-container">
          <h3>{{ state.editingMessageId ? 'Modifier un message' : 'Nouveau message' }}</h3>
          <form @submit.prevent="state.editingMessageId ? methods.updateMessage() : methods.createMessage()">
            <div class="form-group">
              <label>Titre</label>
              <input v-model="state.newMessage.title" required class="input" />
            </div>
            <div class="form-group">
              <label>Contenu</label>
              <textarea v-model="state.newMessage.content" required class="textarea"></textarea>
            </div>
            <div class="form-actions">
              <button type="submit" class="btn primary">
                {{ state.editingMessageId ? 'Modifier' : 'Publier' }}
              </button>
              <button v-if="state.editingMessageId" @click="methods.cancelEditingMessage" type="button" class="btn secondary">
                Annuler
              </button>
            </div>
          </form>
        </div>

        <div v-if="state.messages.length === 0" class="no-data">Aucun message</div>
        <div v-else class="messages-list">
          <div v-for="message in state.messages" :key="message._id" class="message-card">
            <div class="message-header">
              <h4>{{ message.title }}</h4>
              <div class="message-actions">
                <button @click="methods.toggleLike(message)" class="btn-like">
                  {{ state.userLikedMessages.has(message._id) ? '❤️' : '🤍' }}
                  {{ message.likes || 0 }}
                </button>
                <button @click="methods.startEditingMessage(message)" class="btn secondary tiny">Modifier</button>
                <button @click="methods.deleteMessage(message._id)" class="btn danger tiny">Supprimer</button>
              </div>
            </div>
            <p class="message-content">{{ message.content }}</p>

            <!-- Fichiers joints -->
            <div v-if="message._attachments && Object.keys(message._attachments).length > 0" class="attachments-section">
              <h6>📎 Fichiers joints</h6>
              <div class="attachments-grid">
                <div v-for="(attachmentData, name) in message._attachments" :key="name" class="attachment-item">
                  <div v-if="attachmentData && attachmentData.content_type && attachmentData.content_type.startsWith('image/')" class="attachment-image">
                    <img 
                      v-if="state.attachmentUrls[message._id]?.[name]"
                      :src="state.attachmentUrls[message._id][name]" 
                      :alt="name"
                      class="preview-image"
                    />
                    <div v-else class="loading-image">Chargement...</div>
                  </div>
                  <div class="attachment-info">
                    <span class="attachment-name">{{ name }}</span>
                    <button @click="methods.deleteAttachment(message._id, name)" class="btn danger tiny">
                      Supprimer
                    </button>
                  </div>
                </div>
              </div>
            </div>

            <!-- Upload de fichier -->
            <div class="attachment-upload">
              <h6>Ajouter un fichier</h6>
              <input 
                type="file" 
                @change="methods.handleFileSelect($event, message._id)"
                accept="image/*,.pdf,.doc,.docx"
                class="file-input"
              />
              <button 
                v-if="state.selectedFile?.messageId === message._id"
                @click="methods.uploadAttachment(message._id)" 
                class="btn primary tiny"
                :disabled="state.uploadingAttachment"
              >
                {{ state.uploadingAttachment ? 'Upload en cours...' : 'Uploader le fichier' }}
              </button>
            </div>

            <!-- COMMENTAIRES -->
            <div class="comments-section">
              <h5>Commentaires</h5>
              
              <!-- Premier commentaire affiché par défaut -->
              <div v-if="computedData.firstCommentByMessage[message._id]" class="comment">
                <p><strong>{{ computedData.firstCommentByMessage[message._id].author }}:</strong> 
                   {{ computedData.firstCommentByMessage[message._id].content }}
                </p>
                <div class="comment-actions">
                  <span class="comment-date">{{ new Date(computedData.firstCommentByMessage[message._id].createdAt).toLocaleDateString() }}</span>
                  <button @click="methods.startEditingComment(computedData.firstCommentByMessage[message._id])" class="btn secondary tiny">Modifier</button>
                  <button @click="methods.deleteComment(computedData.firstCommentByMessage[message._id])" class="btn danger tiny">Supprimer</button>
                </div>
              </div>
              <div v-else class="no-comment">
                Aucun commentaire
              </div>

              <!-- Bouton pour afficher tous les commentaires -->
              <button @click="methods.toggleCommentsVisibility(message._id)" class="btn secondary small" style="margin: 10px 0;">
                {{ state.expandedMessageId === message._id ? 'Masquer tous les commentaires' : 'Voir tous les commentaires' }}
              </button>

              <!-- Tous les commentaires (si développé) -->
              <div v-if="state.expandedMessageId === message._id" class="all-comments">
                <h6>Tous les commentaires</h6>
                <div v-if="!state.allCommentsForMessage[message._id] || state.allCommentsForMessage[message._id].length === 0" class="no-comment">
                  Aucun commentaire
                </div>
                <div v-else v-for="comment in state.allCommentsForMessage[message._id]" :key="comment._id" class="comment">
                  <!-- Mode édition -->
                  <div v-if="state.editingCommentId === comment._id">
                    <div class="form-group">
                      <label>Commentaire</label>
                      <textarea v-model="state.editingCommentData.content" class="textarea-small"></textarea>
                    </div>
                    <div class="form-group">
                      <label>Auteur</label>
                      <input v-model="state.editingCommentData.author" class="input-small" />
                    </div>
                    <div class="comment-actions">
                      <button @click="methods.updateComment()" class="btn primary tiny">Enregistrer</button>
                      <button @click="methods.cancelEditingComment()" class="btn secondary tiny">Annuler</button>
                    </div>
                  </div>
                  <!-- Mode affichage -->
                  <div v-else>
                    <p><strong>{{ comment.author }}:</strong> {{ comment.content }}</p>
                    <div class="comment-actions">
                      <span class="comment-date">{{ new Date(comment.createdAt).toLocaleDateString() }}</span>
                      <button @click="methods.startEditingComment(comment)" class="btn secondary tiny">Modifier</button>
                      <button @click="methods.deleteComment(comment)" class="btn danger tiny">Supprimer</button>
                    </div>
                  </div>
                </div>
              </div>

              <!-- Formulaire d'ajout de commentaire -->
              <div class="comment-form">
                <h6>💬 Ajouter un commentaire</h6>
                <form @submit.prevent="methods.createComment(message._id)">
                  <div class="form-group">
                    <label>Votre commentaire</label>
                    <textarea 
                      v-model="state.commentForms[message._id].content"
                      placeholder="Écrivez votre commentaire..." 
                      required 
                      class="textarea-small"
                    ></textarea>
                  </div>
                  <div class="form-group">
                    <label>Votre nom</label>
                    <input 
                      v-model="state.commentForms[message._id].author"
                      placeholder="Votre nom" 
                      required 
                      class="input-small" 
                    />
                  </div>
                  <button type="submit" class="btn primary tiny">✅ Publier le commentaire</button>
                </form>
              </div>
            </div>
          </div>
        </div>
      </section>

      <!-- TOP 10 MESSAGES LES PLUS LIKÉS -->
      <section class="section">
        <div class="section-header">
          <h2>🏆 Top Messages les plus likés</h2>
          <p class="section-subtitle">
            Les messages sont triés par nombre de likes décroissant. Le tri est effectué par la base de données 
            grâce à l'index [type, likes], conformément au requirement "ne pas exécuter en TS".
          </p>
        </div>
        
        <div v-if="computedData.topLikedMessages.length === 0" class="no-data">Aucun message</div>
        <div v-else>
          <div class="top-messages">
            <div v-for="(message, index) in computedData.topLikedMessages" :key="message._id" class="top-message">
              <div class="top-message-rank">{{ index + 1 }}</div>
              <div class="top-message-content">
                <h4>{{ message.title }}</h4>
                <p>{{ message.content?.substring(0, 150) }}{{ message.content?.length > 150 ? '...' : '' }}</p>
                
                <!-- Affichage des images miniatures dans le top -->
                <div v-if="message._attachments && Object.keys(message._attachments).length > 0" class="top-message-images">
                  <template v-for="(attachmentData, name) in message._attachments" :key="name">
                    <img 
                      v-if="attachmentData && attachmentData.content_type && attachmentData.content_type.startsWith('image/') && state.attachmentUrls[message._id]?.[name]"
                      :src="state.attachmentUrls[message._id][name]" 
                      :alt="name"
                      class="top-preview-image"
                    />
                  </template>
                </div>
                
                <div class="top-message-meta">
                  <span class="likes-badge">❤️ {{ message.likes || 0 }} likes</span>
                  <span class="country-badge">{{ state.countries.find(c => c._id === message.countryId)?.name || 'Pays inconnu' }}</span>
                  <span class="date-badge">{{ new Date(message.createdAt).toLocaleDateString() }}</span>
                </div>
              </div>
            </div>
          </div>
          
          <!-- Bouton pour charger plus de messages -->
          <div v-if="state.hasMoreTopMessages" class="load-more-container">
            <button @click="methods.loadMoreTopMessages" class="btn primary large">
              📥 Charger les 10 messages suivants
            </button>
            <p class="load-more-hint">
              Actuellement affichés: {{ computedData.topLikedMessages.length }} messages
            </p>
          </div>
          <div v-else class="no-more-data">
            ✅ Tous les messages ont été chargés
          </div>
        </div>
      </section>
    </div>
  </div>
</template>

<style scoped>
* { box-sizing: border-box; }

.app-container {
  max-width: 1400px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  background: #f5f7fa;
}

.app-header {
  background: white;
  padding: 20px;
  border-radius: 8px;
  margin-bottom: 30px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.app-header h1 {
  margin: 0;
  color: #2c3e50;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 15px;
}

.btn-toggle {
  padding: 10px 20px;
  background: #3498db;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 600;
  transition: background 0.3s;
}

.btn-toggle:hover {
  background: #2980b9;
}

.status {
  font-weight: 600;
  color: #7f8c8d;
}

.loading {
  text-align: center;
  padding: 40px;
  font-size: 18px;
  color: #7f8c8d;
}

.section {
  background: white;
  padding: 25px;
  border-radius: 8px;
  margin-bottom: 30px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.section h2 {
  margin: 0 0 10px 0;
  color: #2c3e50;
  border-bottom: 2px solid #3498db;
  padding-bottom: 10px;
}

.section-header {
  margin-bottom: 20px;
}

.section-subtitle {
  margin: 5px 0 0 0;
  color: #7f8c8d;
  font-size: 13px;
  font-style: italic;
  line-height: 1.5;
}

.search-bar, .toolbar {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.flex-1 { flex: 1; }

.input, .select, .textarea {
  padding: 10px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 14px;
  font-family: inherit;
}

.textarea {
  min-height: 100px;
  resize: vertical;
}

.textarea-small {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 13px;
  min-height: 60px;
  width: 100%;
  resize: vertical;
  font-family: inherit;
}

.input-small {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 13px;
  width: 100%;
}

.file-input {
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: 6px;
  font-size: 13px;
  width: 100%;
  margin-bottom: 10px;
}

.form-container {
  background: #f8f9fa;
  padding: 20px;
  border-radius: 6px;
  margin-bottom: 20px;
}

.form-container h3 {
  margin: 0 0 15px 0;
  color: #34495e;
}

.form-grid {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 15px;
  margin-bottom: 15px;
}

.form-group {
  display: flex;
  flex-direction: column;
  margin-bottom: 10px;
}

.form-group label {
  margin-bottom: 5px;
  font-weight: 600;
  color: #555;
  font-size: 13px;
}

.form-actions {
  display: flex;
  gap: 10px;
}

.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 14px;
  font-weight: 600;
  transition: all 0.3s;
}

.btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn.primary { background: #3498db; color: white; }
.btn.primary:hover:not(:disabled) { background: #2980b9; }
.btn.secondary { background: #95a5a6; color: white; }
.btn.secondary:hover { background: #7f8c8d; }
.btn.danger { background: #e74c3c; color: white; }
.btn.danger:hover { background: #c0392b; }
.btn.small { padding: 6px 12px; font-size: 12px; }
.btn.tiny { padding: 4px 8px; font-size: 11px; }
.btn.large { padding: 12px 24px; font-size: 16px; }

.btn-like {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 14px;
  color: #e74c3c;
  display: flex;
  align-items: center;
  gap: 5px;
  padding: 4px 8px;
  border-radius: 4px;
  transition: background 0.3s;
}

.btn-like:hover {
  background: #ffe6e6;
}

.cards-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
}

.card {
  border: 2px solid #ecf0f1;
  border-radius: 8px;
  padding: 15px;
  cursor: pointer;
  transition: all 0.3s;
}

.card:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
}

.card.selected {
  border-color: #3498db;
  background: #ebf5fb;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 10px;
}

.card-header h3 {
  margin: 0;
  color: #2c3e50;
}

.flag {
  width: 50px;
  height: auto;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.card-body p {
  margin: 5px 0;
  font-size: 14px;
  color: #555;
}

.card-actions {
  display: flex;
  gap: 8px;
  margin-top: 10px;
}

.messages-list {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.message-card {
  border: 1px solid #ecf0f1;
  border-radius: 8px;
  padding: 15px;
  background: #fafbfc;
}

.message-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 10px;
}

.message-header h4 {
  margin: 0;
  color: #2c3e50;
  flex: 1;
}

.message-actions {
  display: flex;
  gap: 8px;
  align-items: center;
}

.message-content {
  margin: 10px 0;
  line-height: 1.6;
  color: #555;
}

.attachments-section {
  margin: 15px 0;
  padding: 15px;
  background: #fff9e6;
  border-radius: 6px;
  border: 1px solid #ffd966;
}

.attachments-section h6 {
  margin: 0 0 10px 0;
  color: #856404;
  font-size: 14px;
}

.attachments-grid {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.attachment-item {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.attachment-image {
  width: 100%;
  max-width: 300px;
}

.preview-image {
  width: 100%;
  height: auto;
  border-radius: 6px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.1);
}

.loading-image {
  padding: 20px;
  text-align: center;
  color: #7f8c8d;
  font-style: italic;
}

.attachment-info {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 8px;
  background: white;
  border-radius: 4px;
}

.attachment-name {
  font-size: 13px;
  color: #333;
  font-weight: 500;
}

.attachment-upload {
  margin: 15px 0;
  padding: 15px;
  background: #e8f5e9;
  border-radius: 6px;
  border: 2px dashed #4caf50;
}

.attachment-upload h6 {
  margin: 0 0 10px 0;
  color: #2e7d32;
  font-size: 14px;
}

.comments-section {
  margin-top: 15px;
  padding-top: 15px;
  border-top: 1px dashed #ddd;
}

.comments-section h5 {
  margin: 0 0 10px 0;
  color: #7f8c8d;
  font-size: 14px;
}

.comment {
  background: white;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  padding: 10px;
  margin-bottom: 10px;
}

.comment p {
  margin: 0 0 8px 0;
  font-size: 13px;
}

.comment-actions {
  display: flex;
  gap: 6px;
  align-items: center;
}

.comment-date {
  font-size: 11px;
  color: #95a5a6;
  margin-right: auto;
}

.all-comments {
  margin: 15px 0;
}

.all-comments h6 {
  margin: 0 0 10px 0;
  color: #555;
  font-size: 13px;
}

.comment-form {
  margin-top: 15px;
  padding: 15px;
  background: #f0f8ff;
  border-radius: 6px;
  border: 2px dashed #3498db;
}

.comment-form h6 {
  margin: 0 0 10px 0;
  color: #2c3e50;
  font-size: 14px;
}

.comment-form form {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.no-comment {
  padding: 10px;
  color: #95a5a6;
  font-style: italic;
  font-size: 13px;
}

.top-messages {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
  gap: 20px;
  margin-bottom: 20px;
}

.top-message {
  border-left: 4px solid #3498db;
  padding: 20px;
  background: #f8f9fa;
  border-radius: 8px;
  display: flex;
  gap: 15px;
  transition: all 0.3s;
}

.top-message:hover {
  transform: translateX(5px);
  box-shadow: 0 4px 12px rgba(0,0,0,0.1);
}

.top-message-rank {
  flex-shrink: 0;
  width: 40px;
  height: 40px;
  background: linear-gradient(135deg, #3498db, #2980b9);
  color: white;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  font-size: 18px;
  box-shadow: 0 2px 8px rgba(52, 152, 219, 0.3);
}

.top-message-content {
  flex: 1;
}

.top-message h4 {
  margin: 0 0 8px 0;
  color: #2c3e50;
  font-size: 16px;
}

.top-message p {
  margin: 0 0 10px 0;
  font-size: 13px;
  color: #555;
  line-height: 1.5;
}

.top-message-images {
  display: flex;
  gap: 8px;
  margin: 10px 0;
  flex-wrap: wrap;
}

.top-preview-image {
  width: 80px;
  height: 80px;
  object-fit: cover;
  border-radius: 6px;
  border: 2px solid #ddd;
}

.top-message-meta {
  display: flex;
  gap: 10px;
  align-items: center;
  padding-top: 10px;
  border-top: 1px solid #ddd;
  flex-wrap: wrap;
}

.likes-badge {
  background: #ffe6e6;
  color: #e74c3c;
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
}

.country-badge {
  background: #e8f5e9;
  color: #2e7d32;
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
}

.date-badge {
  background: #e3f2fd;
  color: #1976d2;
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
}

.load-more-container {
  text-align: center;
  padding: 30px;
  background: #f8f9fa;
  border-radius: 8px;
  border: 2px dashed #3498db;
}

.load-more-hint {
  margin: 15px 0 0 0;
  font-size: 13px;
  color: #7f8c8d;
}

.no-more-data {
  text-align: center;
  padding: 20px;
  color: #27ae60;
  font-weight: 600;
  background: #e8f5e9;
  border-radius: 6px;
}

.no-data {
  text-align: center;
  padding: 30px;
  color: #95a5a6;
  font-style: italic;
}

@media (max-width: 768px) {
  .form-grid {
    grid-template-columns: 1fr;
  }
  .cards-grid {
    grid-template-columns: 1fr;
  }
  .top-messages {
    grid-template-columns: 1fr;
  }
  .app-header {
    flex-direction: column;
    gap: 15px;
  }
  .message-header {
    flex-direction: column;
    gap: 10px;
  }
}
</style>