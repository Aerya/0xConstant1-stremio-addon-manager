<script setup>
import { ref, computed } from 'vue'
import draggable from 'vuedraggable'
import AddonItem from './AddonItem.vue'
import Authentication from './Authentication.vue'
import DynamicForm from './DynamicForm.vue'

const stremioAPIBase = "https://api.strem.io/api/"
let stremioAuthKey = ref('');
let addons = ref([])
let loadAddonsButtonText = ref('Load Addons')
const isLoadingAddons = ref(false)
const isSyncing = ref(false)
const isAuthenticated = ref(false)
const isLocked = computed(() => !isAuthenticated.value)

let isEditModalVisible = ref(false);
let currentManifest = ref({});
let currentEditIdx = ref(null);

// Feature toggles for manifest manipulation
const shouldRemoveSearchArtifacts = ref(false)
const shouldRemoveStandardCatalogs = ref(false)
const shouldRemoveMetaResource = ref(false)

// Patch status tracking
const isSearchArtifactsPatched = ref(false)
const isStandardCatalogsPatched = ref(false)
const isMetaResourcePatched = ref(false)

// Restore functionality state
const backupFile = ref(null)
const backupFileName = ref('')
const isBackupValid = ref(false)
const isRestoring = ref(false)

// Addon management state
const isManageAddonVisible = ref(false)
const newAddonManifest = ref({
    name: '',
    description: '',
    logo: '',
    background: '',
    catalogs: []
})

// Reset Cinemeta state
const isResettingCinemeta = ref(false)

// Generic addon reset state
const isResettingAddon = ref({})

const addonsCount = computed(() => Array.isArray(addons.value) ? addons.value.length : 0)

/**
 * Remove Cinemeta search catalogs and the 'search' extra from top catalogs.
 * - Deletes catalogs with id 'cinemeta.search' for types movie and series
 * - Removes extra entry with name 'search' from 'top' movie and series catalogs
 * Returns a summary of changes for logging.
 */
function removeCinemetaSearchArtifacts() {
    const cinemetaIndex = addons.value.findIndex((addon) => addon && addon.manifest && addon.manifest.name === 'Cinemeta')
    if (cinemetaIndex === -1) {
        return {
            removedSearchMovieCatalog: false,
            removedSearchSeriesCatalog: false,
            removedTopMovieSearchExtra: false,
            removedTopSeriesSearchExtra: false,
        }
    }

    const cinemetaAddon = addons.value[cinemetaIndex]
    const manifest = cinemetaAddon.manifest || {}
    const originalCatalogs = Array.isArray(manifest.catalogs) ? manifest.catalogs : []

    const hadSearchMovie = originalCatalogs.some((c) => c && c.id === 'cinemeta.search' && c.type === 'movie')
    const hadSearchSeries = originalCatalogs.some((c) => c && c.id === 'cinemeta.search' && c.type === 'series')

    // Remove the search catalogs entirely
    let updatedCatalogs = originalCatalogs.filter((c) => !(c && c.id === 'cinemeta.search' && (c.type === 'movie' || c.type === 'series')))

    // Helper to remove 'search' from extra array
    function stripSearchExtra(catalog) {
        if (!catalog || !Array.isArray(catalog.extra)) return false
        const before = catalog.extra.length
        catalog.extra = catalog.extra.filter((e) => !(e && e.name === 'search'))
        return catalog.extra.length !== before
    }

    // Remove 'search' from top movie and top series extras
    const topMovie = updatedCatalogs.find((c) => c && c.id === 'top' && c.type === 'movie')
    const topSeries = updatedCatalogs.find((c) => c && c.id === 'top' && c.type === 'series')
    const removedTopMovieSearchExtra = stripSearchExtra(topMovie)
    const removedTopSeriesSearchExtra = stripSearchExtra(topSeries)

    // Persist changes back to the reactive state
    addons.value[cinemetaIndex].manifest = {
        ...manifest,
        catalogs: updatedCatalogs,
    }

    return {
        removedSearchMovieCatalog: hadSearchMovie,
        removedSearchSeriesCatalog: hadSearchSeries,
        removedTopMovieSearchExtra,
        removedTopSeriesSearchExtra,
    }
}

/**
 * Remove specific Cinemeta catalogs by id/type mapping:
 * - Popular => id 'top' for movie and series
 * - New => id 'year' for movie and series
 * - Featured => id 'imdbRating' for movie and series
 * 
 * Conditional behavior:
 * - If both 'Remove Cinemeta Search' AND 'Remove Cinemeta Catalogs' are ON: remove all catalogs
 * - If only 'Remove Cinemeta Catalogs' is ON but 'Remove Cinemeta Search' is OFF: 
 *   modify Popular catalogs (add isRequired: true to search extra) and remove others
 * 
 * Returns a summary of which targets were removed.
 */
function removeCinemetaStandardCatalogs() {
    const cinemetaIndex = addons.value.findIndex((addon) => addon && addon.manifest && addon.manifest.name === 'Cinemeta')
    if (cinemetaIndex === -1) {
        return {
            removedPopularMovie: false,
            removedPopularSeries: false,
            removedNewMovie: false,
            removedNewSeries: false,
            removedFeaturedMovie: false,
            removedFeaturedSeries: false,
        }
    }

    const manifest = addons.value[cinemetaIndex].manifest || {}
    const originalCatalogs = Array.isArray(manifest.catalogs) ? manifest.catalogs : []

    const isPopularMovie = (c) => c && c.id === 'top' && c.type === 'movie'
    const isPopularSeries = (c) => c && c.id === 'top' && c.type === 'series'
    const isNewMovie = (c) => c && c.id === 'year' && c.type === 'movie'
    const isNewSeries = (c) => c && c.id === 'year' && c.type === 'series'
    const isFeaturedMovie = (c) => c && c.id === 'imdbRating' && c.type === 'movie'
    const isFeaturedSeries = (c) => c && c.id === 'imdbRating' && c.type === 'series'

    const hadPopularMovie = originalCatalogs.some(isPopularMovie)
    const hadPopularSeries = originalCatalogs.some(isPopularSeries)
    const hadNewMovie = originalCatalogs.some(isNewMovie)
    const hadNewSeries = originalCatalogs.some(isNewSeries)
    const hadFeaturedMovie = originalCatalogs.some(isFeaturedMovie)
    const hadFeaturedSeries = originalCatalogs.some(isFeaturedSeries)

    // Check conditional behavior: if Remove Cinemeta Catalogs is ON but Remove Cinemeta Search is OFF
    const shouldModifyPopularInsteadOfRemove = shouldRemoveStandardCatalogs.value && !shouldRemoveSearchArtifacts.value

    let updatedCatalogs
    let actuallyRemovedPopularMovie = false
    let actuallyRemovedPopularSeries = false

    if (shouldModifyPopularInsteadOfRemove) {
        // Modify Popular catalogs (add isRequired: true to search extra) and remove others
        updatedCatalogs = originalCatalogs.map((catalog) => {
            if (isPopularMovie(catalog) || isPopularSeries(catalog)) {
                // Modify the search extra to add isRequired: true
                const modifiedCatalog = { ...catalog }
                if (Array.isArray(modifiedCatalog.extra)) {
                    modifiedCatalog.extra = modifiedCatalog.extra.map((extra) => {
                        if (extra && extra.name === 'search') {
                            return { ...extra, isRequired: true }
                        }
                        return extra
                    })
                }
                return modifiedCatalog
            }
            return catalog
        }).filter((c) => !(
            // Still remove New and Featured catalogs completely
            isNewMovie(c) ||
            isNewSeries(c) ||
            isFeaturedMovie(c) ||
            isFeaturedSeries(c)
        ))
        
        // Popular catalogs are modified, not removed
        actuallyRemovedPopularMovie = false
        actuallyRemovedPopularSeries = false
    } else {
        // Original behavior - remove all standard catalogs including Popular ones
        updatedCatalogs = originalCatalogs.filter((c) => !(
            isPopularMovie(c) ||
            isPopularSeries(c) ||
            isNewMovie(c) ||
            isNewSeries(c) ||
            isFeaturedMovie(c) ||
            isFeaturedSeries(c)
        ))
        
        actuallyRemovedPopularMovie = hadPopularMovie
        actuallyRemovedPopularSeries = hadPopularSeries
    }

    addons.value[cinemetaIndex].manifest = {
        ...manifest,
        catalogs: updatedCatalogs,
    }

    return {
        removedPopularMovie: actuallyRemovedPopularMovie,
        removedPopularSeries: actuallyRemovedPopularSeries,
        removedNewMovie: hadNewMovie,
        removedNewSeries: hadNewSeries,
        removedFeaturedMovie: hadFeaturedMovie,
        removedFeaturedSeries: hadFeaturedSeries,
    }
}

/**
 * removeMeta: Remove the 'meta' entry from Cinemeta's manifest.resources
 * Returns whether 'meta' was present and removed.
 */
function removeMeta() {
    const cinemetaIndex = addons.value.findIndex((addon) => addon && addon.manifest && addon.manifest.name === 'Cinemeta')
    if (cinemetaIndex === -1) {
        return { removedMeta: false }
    }
    const manifest = addons.value[cinemetaIndex].manifest || {}
    const resources = Array.isArray(manifest.resources) ? manifest.resources : []
    const hadMeta = resources.includes('meta')
    if (!hadMeta) {
        return { removedMeta: false }
    }
    const updatedResources = resources.filter((r) => r !== 'meta')
    addons.value[cinemetaIndex].manifest = {
        ...manifest,
        resources: updatedResources,
    }
    return { removedMeta: true }
}

/**
 * Detect if Cinemeta search artifacts have been removed
 * Returns true if the search catalogs and extras are missing (patch applied)
 */
function detectSearchArtifactsPatched() {
    const cinemetaIndex = addons.value.findIndex((addon) => addon && addon.manifest && addon.manifest.name === 'Cinemeta')
    if (cinemetaIndex === -1) {
        return false // No Cinemeta addon found
    }

    const manifest = addons.value[cinemetaIndex].manifest || {}
    const catalogs = Array.isArray(manifest.catalogs) ? manifest.catalogs : []

    // Check if search catalogs are missing
    const hasSearchMovie = catalogs.some((c) => c && c.id === 'cinemeta.search' && c.type === 'movie')
    const hasSearchSeries = catalogs.some((c) => c && c.id === 'cinemeta.search' && c.type === 'series')

    // Check if 'search' extra is missing from top catalogs
    const topMovie = catalogs.find((c) => c && c.id === 'top' && c.type === 'movie')
    const topSeries = catalogs.find((c) => c && c.id === 'top' && c.type === 'series')
    
    const topMovieHasSearchExtra = topMovie && Array.isArray(topMovie.extra) && topMovie.extra.some((e) => e && e.name === 'search')
    const topSeriesHasSearchExtra = topSeries && Array.isArray(topSeries.extra) && topSeries.extra.some((e) => e && e.name === 'search')

    // Patch is applied if search catalogs are missing AND search extras are missing from top catalogs
    return !hasSearchMovie && !hasSearchSeries && !topMovieHasSearchExtra && !topSeriesHasSearchExtra
}

/**
 * Detect if Cinemeta standard catalogs have been removed or modified
 * Returns true if:
 * - All standard catalogs are missing (both toggles ON), OR
 * - Popular catalogs are modified (search extra has isRequired: true) and New/Featured are removed
 *   (only 'Remove Cinemeta Catalogs' ON but 'Remove Cinemeta Search' OFF)
 */
function detectStandardCatalogsPatched() {
    const cinemetaIndex = addons.value.findIndex((addon) => addon && addon.manifest && addon.manifest.name === 'Cinemeta')
    if (cinemetaIndex === -1) {
        return false // No Cinemeta addon found
    }

    const manifest = addons.value[cinemetaIndex].manifest || {}
    const catalogs = Array.isArray(manifest.catalogs) ? manifest.catalogs : []

    // Check if standard catalogs exist
    const hasPopularMovie = catalogs.some((c) => c && c.id === 'top' && c.type === 'movie')
    const hasPopularSeries = catalogs.some((c) => c && c.id === 'top' && c.type === 'series')
    const hasNewMovie = catalogs.some((c) => c && c.id === 'year' && c.type === 'movie')
    const hasNewSeries = catalogs.some((c) => c && c.id === 'year' && c.type === 'series')
    const hasFeaturedMovie = catalogs.some((c) => c && c.id === 'imdbRating' && c.type === 'movie')
    const hasFeaturedSeries = catalogs.some((c) => c && c.id === 'imdbRating' && c.type === 'series')

    // Check if Popular catalogs have been modified (search extra has isRequired: true)
    const popularMovieCatalog = catalogs.find((c) => c && c.id === 'top' && c.type === 'movie')
    const popularSeriesCatalog = catalogs.find((c) => c && c.id === 'top' && c.type === 'series')
    
    const isPopularMovieModified = popularMovieCatalog && Array.isArray(popularMovieCatalog.extra) && 
        popularMovieCatalog.extra.some((e) => e && e.name === 'search' && e.isRequired === true)
    const isPopularSeriesModified = popularSeriesCatalog && Array.isArray(popularSeriesCatalog.extra) && 
        popularSeriesCatalog.extra.some((e) => e && e.name === 'search' && e.isRequired === true)

    // Scenario 1: All standard catalogs are removed (both toggles ON)
    const allCatalogsRemoved = !hasPopularMovie && !hasPopularSeries && !hasNewMovie && !hasNewSeries && !hasFeaturedMovie && !hasFeaturedSeries

    // Scenario 2: Popular catalogs are modified and New/Featured are removed (only 'Remove Cinemeta Catalogs' ON)
    const popularModifiedAndOthersRemoved = 
        hasPopularMovie && hasPopularSeries && 
        isPopularMovieModified && isPopularSeriesModified &&
        !hasNewMovie && !hasNewSeries && !hasFeaturedMovie && !hasFeaturedSeries

    // Patch is applied if either scenario is true
    return allCatalogsRemoved || popularModifiedAndOthersRemoved
}

/**
 * Detect if Cinemeta meta resource has been removed
 * Returns true if 'meta' is missing from resources (patch applied)
 */
function detectMetaResourcePatched() {
    const cinemetaIndex = addons.value.findIndex((addon) => addon && addon.manifest && addon.manifest.name === 'Cinemeta')
    if (cinemetaIndex === -1) {
        return false // No Cinemeta addon found
    }

    const manifest = addons.value[cinemetaIndex].manifest || {}
    const resources = Array.isArray(manifest.resources) ? manifest.resources : []
    
    // Patch is applied if 'meta' is not in resources
    return !resources.includes('meta')
}

/**
 * Check all patches and update the reactive state accordingly
 */
function detectAppliedPatches() {
    if (!Array.isArray(addons.value) || addons.value.length === 0) {
        return
    }

    try {
        isSearchArtifactsPatched.value = detectSearchArtifactsPatched()
        isStandardCatalogsPatched.value = detectStandardCatalogsPatched()
        isMetaResourcePatched.value = detectMetaResourcePatched()

        // Update toggles to reflect current state
        shouldRemoveSearchArtifacts.value = isSearchArtifactsPatched.value
        shouldRemoveStandardCatalogs.value = isStandardCatalogsPatched.value
        shouldRemoveMetaResource.value = isMetaResourcePatched.value

        console.log('Patch detection results:', {
            searchArtifacts: isSearchArtifactsPatched.value,
            standardCatalogs: isStandardCatalogsPatched.value,
            metaResource: isMetaResourcePatched.value
        })
    } catch (e) {
        console.error('Error detecting applied patches:', e)
    }
}

function loadUserAddons() {
    const key = stremioAuthKey.value
    if (!key) {
        console.error('No auth key provided')
        return
    }

    loadAddonsButtonText.value = 'Loading...'
    isLoadingAddons.value = true
    console.log('Loading addons...')

    const url = `${stremioAPIBase}addonCollectionGet`
    fetch(url, {
        method: 'POST',
        body: JSON.stringify({
            type: 'AddonCollectionGet',
            authKey: key,
            update: true,
        })
    }).then((resp) => {
        resp.json().then((data) => {
            console.log(data)
            if (!("result" in data) || data.result == null) {
                console.error("Failed to fetch user addons: ", data)
                alert('Failed to fetch user addons - are you sure you pasted the correct Stremio AuthKey?')
                return
            }
            addons.value = data.result.addons
            // Detect already applied patches and update toggles
            detectAppliedPatches()
        })
    }).catch((error) => {
        console.error('Error fetching user addons', error)
    }).finally(() => {
        loadAddonsButtonText.value = 'Load Addons'
        isLoadingAddons.value = false
    })
}

function syncUserAddons() {
    const key = stremioAuthKey.value
    if (!key) {
        console.error('No auth key provided')
        return
    }
    console.log('Syncing addons...')
    isSyncing.value = true

    // Apply selected manifest manipulations before syncing
    try {
        if (shouldRemoveSearchArtifacts.value) {
            const result = removeCinemetaSearchArtifacts()
            console.log('Applied removeCinemetaSearchArtifacts:', result)
        }
        if (shouldRemoveStandardCatalogs.value) {
            const result = removeCinemetaStandardCatalogs()
            console.log('Applied removeCinemetaStandardCatalogs:', result)
        }
        if (shouldRemoveMetaResource.value) {
            const result = removeMeta()
            console.log('Applied removeMeta:', result)
        }
    } catch (e) {
        console.error('Error applying selected manifest manipulations:', e)
    }

    const url = `${stremioAPIBase}addonCollectionSet`
    fetch(url, {
        method: 'POST',
        body: JSON.stringify({
            type: 'AddonCollectionSet',
            authKey: key,
            addons: addons.value,
        })
    }).then((resp) => {
        resp.json().then((data) => {
            if (!("result" in data) || data.result == null) {
                console.error("Sync failed: ", data)
                alert('Sync failed if unknown error')
                return
            } else if (!data.result.success) {
                alert("Failed to sync addons: " + data.result.error)
            } else {
                console.log("Sync complete: + ", data)
                
                // Update patch status flags to match successfully synced toggle states
                isSearchArtifactsPatched.value = shouldRemoveSearchArtifacts.value
                isStandardCatalogsPatched.value = shouldRemoveStandardCatalogs.value
                isMetaResourcePatched.value = shouldRemoveMetaResource.value
                
                // Run detection to verify patches were applied correctly (extra validation)
                detectAppliedPatches()
                
                alert('Sync complete!')
            }
        })
    }).catch((error) => {
        alert("Error syncing addons: " + error)
        console.error('Error fetching user addons', error)
    }).finally(() => {
        isSyncing.value = false
    })
}

function removeAddon(idx) {
    addons.value.splice(idx, 1)
}

function getNestedObjectProperty(obj, path, defaultValue = null) {
    try {
        return path.split('.').reduce((acc, part) => acc && acc[part], obj)
    } catch (e) {
        return defaultValue
    }
}

function setAuthKey(authKey) {
    stremioAuthKey.value = authKey
    console.log('AuthKey set to: ', stremioAuthKey.value)
}

function onAuthSuccess(authKey) {
    // Ensure our local state has the latest key, then auto-load addons
    setAuthKey(authKey)
    isAuthenticated.value = true
    console.log('Authentication successful, auto-loading addons...')
    loadUserAddons()
}

function openEditModal(idx) {
    isEditModalVisible.value = true;
    currentEditIdx.value = idx;
    currentManifest.value = { ...addons.value[idx].manifest };
    document.body.classList.add('modal-open');
}

function closeEditModal() {
    isEditModalVisible.value = false;
    currentManifest.value = {};
    currentEditIdx.value = null;
    document.body.classList.remove('modal-open');
}

function saveManifestEdit(updatedManifest) {
    try {
        addons.value[currentEditIdx.value].manifest = updatedManifest;
        closeEditModal();
    } catch (e) {
        alert('Failed to update manifest');
    }
}

function downloadBackup() {
    if (!addons.value || addons.value.length === 0) {
        alert('No configuration data available to backup');
        return;
    }

    try {
        // Only include the original addon data from the server
        const backupData = JSON.parse(JSON.stringify(addons.value)); // Deep clone to ensure original data

        const dataStr = JSON.stringify(backupData, null, 2);
        const dataBlob = new Blob([dataStr], { type: 'application/json' });
        
        const url = URL.createObjectURL(dataBlob);
        const link = document.createElement('a');
        link.href = url;
        link.download = `stremio-addons-backup-${new Date().toISOString().split('T')[0]}.json`;
        
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
        
        URL.revokeObjectURL(url);
        
        console.log('Backup downloaded successfully');
    } catch (e) {
        console.error('Error creating backup:', e);
        alert('Failed to create backup file');
    }
}

function loadBackupFile() {
    const input = document.createElement('input');
    input.type = 'file';
    input.accept = '.json';
    
    input.onchange = (event) => {
        const file = event.target.files[0];
        if (!file) return;
        
        const reader = new FileReader();
        reader.onload = (e) => {
            try {
                let jsonData = JSON.parse(e.target.result);
                
                // Check if it's the alternative format (object with "addons" property)
                if (!Array.isArray(jsonData) && jsonData && typeof jsonData === 'object' && Array.isArray(jsonData.addons)) {
                    console.log('Detected Sidekick backup format');
                    jsonData = jsonData.addons; // Extract the addons array
                }
                
                // Validate that it's an array and has addon-like structure
                if (!Array.isArray(jsonData)) {
                    throw new Error('Backup file must contain an array of addons');
                }
                
                // Basic validation - check if items have manifest property
                const hasValidStructure = jsonData.every(item => 
                    item && typeof item === 'object' && item.manifest
                );
                
                if (!hasValidStructure) {
                    throw new Error('Invalid addon structure in backup file');
                }
                
                // Store the validated data
                backupFile.value = jsonData;
                backupFileName.value = file.name;
                isBackupValid.value = true;
                
                console.log(`Loaded backup file: ${file.name} with ${jsonData.length} addons`);
            } catch (error) {
                console.error('Error parsing backup file:', error);
                alert(`Invalid backup file: ${error.message}`);
                
                // Reset state on error
                backupFile.value = null;
                backupFileName.value = '';
                isBackupValid.value = false;
            }
        };
        
        reader.readAsText(file);
    };
    
    input.click();
}

function restoreFromBackup() {
    if (!backupFile.value || !isBackupValid.value) {
        alert('No valid backup file loaded');
        return;
    }
    
    const key = stremioAuthKey.value;
    if (!key) {
        console.error('No auth key provided');
        return;
    }
    
    console.log('Restoring addons from backup...');
    isRestoring.value = true;
    
    const url = `${stremioAPIBase}addonCollectionSet`;
    fetch(url, {
        method: 'POST',
        body: JSON.stringify({
            type: 'AddonCollectionSet',
            authKey: key,
            addons: backupFile.value, // Use backup data instead of current addons
        })
    }).then((resp) => {
        resp.json().then((data) => {
            if (!("result" in data) || data.result == null) {
                console.error("Restore failed: ", data);
                alert('Restore failed with unknown error');
                return;
            } else if (!data.result.success) {
                alert("Failed to restore addons: " + data.result.error);
            } else {
                console.log("Restore complete: ", data);
                alert(`Successfully restored ${backupFile.value.length} addons from ${backupFileName.value}!`);
                
                // Update local state with restored data
                addons.value = JSON.parse(JSON.stringify(backupFile.value))
                
                // Detect already applied patches after restore
                detectAppliedPatches();
                
                // Reset backup state after successful restore
                backupFile.value = null;
                backupFileName.value = '';
                isBackupValid.value = false;
            }
        });
    }).catch((error) => {
        alert("Error restoring addons: " + error);
        console.error('Error restoring addons', error);
    }).finally(() => {
        isRestoring.value = false;
    });
}

function toggleAddonManagement() {
    isManageAddonVisible.value = !isManageAddonVisible.value;
    if (!isManageAddonVisible.value) {
        resetNewAddonForm();
    }
}

function resetNewAddonForm() {
    newAddonManifest.value = {
        name: '',
        description: '',
        logo: '',
        background: '',
        catalogs: []
    };
}

function saveNewAddon(manifest) {
    try {
        // Create a new addon object in the format expected by Stremio
        const newAddon = {
            manifest: { ...manifest },
            transportUrl: '', // Will be set by user or generated
            flags: {}
        };
        
        // Add to the addons list
        if (!Array.isArray(addons.value)) {
            addons.value = [];
        }
        addons.value.push(newAddon);
        
        console.log('New addon created:', newAddon);
        alert(`Addon "${manifest.name}" created successfully! Remember to sync to apply changes.`);
        
        // Reset form and hide management UI
        resetNewAddonForm();
        isManageAddonVisible.value = false;
    } catch (e) {
        console.error('Error creating new addon:', e);
        alert('Failed to create new addon');
    }
}

/**
 * Normalizes a manifest URL to ensure it's a valid HTTPS manifest.json endpoint
 * Based on Go implementation with enhancements for JavaScript
 * 
 * Handles:
 * - stremio:// to https:// scheme conversion
 * - /configure to /manifest.json path conversion  
 * - Appending /manifest.json if missing
 * - Cleaning query parameters and fragments
 * - URL validation and error handling
 * 
 * @param {string} manifestUrl - The URL to normalize
 * @returns {string} - The normalized manifest.json URL
 * @throws {Error} - If URL is invalid or uses unsupported protocol
 */
function normalizeManifestUrl(manifestUrl) {
    try {
        // Handle stremio:// scheme by converting to https://
        if (manifestUrl.startsWith('stremio://')) {
            manifestUrl = manifestUrl.replace('stremio://', 'https://');
        }
        
        // Parse the URL for proper manipulation
        const url = new URL(manifestUrl);
        
        // Convert any remaining stremio scheme to https
        if (url.protocol === 'stremio:') {
            url.protocol = 'https:';
        }
        
        // Ensure we have a valid HTTP/HTTPS protocol
        if (url.protocol !== 'http:' && url.protocol !== 'https:') {
            throw new Error(`Unsupported protocol: ${url.protocol}. Only http:// and https:// are supported.`);
        }
        
        // Validate hostname is not empty
        if (!url.hostname) {
            throw new Error('Invalid URL: missing hostname');
        }
        
        // Handle /configure paths by replacing with /manifest.json
        if (url.pathname.endsWith('/configure')) {
            url.pathname = url.pathname.replace(/\/configure$/, '/manifest.json');
        }
        // Handle /configure/ paths
        else if (url.pathname.endsWith('/configure/')) {
            url.pathname = url.pathname.replace(/\/configure\/$/, '/manifest.json');
        }
        // If path doesn't end with manifest.json, append it
        else if (!url.pathname.endsWith('/manifest.json')) {
            // Ensure proper path separator
            url.pathname = url.pathname.endsWith('/') 
                ? url.pathname + 'manifest.json' 
                : url.pathname + '/manifest.json';
        }
        
        // Clear query parameters and fragments for manifest.json URLs
        // (they're not typically needed for manifest fetching)
        url.search = '';
        url.hash = '';
        
        return url.toString();
    } catch (error) {
        // If URL parsing fails, fall back to string manipulation
        console.warn(`Failed to parse URL "${manifestUrl}", falling back to string manipulation:`, error);
        
        let normalizedUrl = manifestUrl;
        
        // Handle stremio:// scheme conversion
        if (normalizedUrl.startsWith('stremio://')) {
            normalizedUrl = normalizedUrl.replace('stremio://', 'https://');
        }
        
        // Handle /configure paths
        if (normalizedUrl.includes('/configure')) {
            normalizedUrl = normalizedUrl.replace(/\/configure\/?$/, '/manifest.json');
        }
        // Ensure manifest.json ending
        else if (!normalizedUrl.endsWith('/manifest.json')) {
            normalizedUrl = normalizedUrl.endsWith('/') 
                ? normalizedUrl + 'manifest.json' 
                : normalizedUrl + '/manifest.json';
        }
        
        // Clean up query parameters and fragments from fallback URL
        normalizedUrl = normalizedUrl.split('?')[0].split('#')[0];
        
        return normalizedUrl;
    }
}

/**
 * Shared validation helper for addon reset operations
 * Validates the fetched manifest and current addon state before applying changes
 */
function validateAddonResetSafety(expectedAddon, fetchedManifest, currentIndex, additionalChecks = {}) {
    // Validate the fetched manifest structure
    if (!fetchedManifest || typeof fetchedManifest !== 'object') {
        throw new Error(`Invalid manifest received for ${expectedAddon.name}`);
    }
    
    // Safety check: verify the fetched manifest is for the expected addon
    if (fetchedManifest.name !== expectedAddon.name) {
        throw new Error(`Manifest name mismatch: expected "${expectedAddon.name}", but received manifest for "${fetchedManifest.name || 'unnamed addon'}"`);
    }
    
    // Safety check: verify the addon at this index is still the same one we started with
    const currentAddon = addons.value[currentIndex];
    if (!currentAddon || !currentAddon.manifest || currentAddon.manifest.name !== expectedAddon.name) {
        throw new Error(`Addon mismatch: expected "${expectedAddon.name}" at index ${currentIndex}, but found "${currentAddon?.manifest?.name || 'unknown'}"`);
    }
    
    // Additional safety: verify the transportUrl hasn't changed
    if (currentAddon.transportUrl !== expectedAddon.transportUrl) {
        throw new Error(`Transport URL mismatch for "${expectedAddon.name}": expected "${expectedAddon.transportUrl}", but found "${currentAddon.transportUrl}"`);
    }
    
    // Apply any additional custom checks
    if (additionalChecks.transportUrlContains) {
        if (!currentAddon.transportUrl || !currentAddon.transportUrl.includes(additionalChecks.transportUrlContains)) {
            throw new Error(`Addon at index ${currentIndex} doesn't appear to be ${expectedAddon.name} (transportUrl: "${currentAddon.transportUrl}")`);
        }
    }
    
    return currentAddon;
}

/**
 * Generic function to reset any addon to its original state by fetching 
 * the unmodified manifest from its transportUrl
 */
function resetAddon(addonIndex) {
    const key = stremioAuthKey.value;
    if (!key) {
        console.error('No auth key provided');
        alert('Authentication required to reset addon');
        return;
    }

    if (addonIndex < 0 || addonIndex >= addons.value.length) {
        alert('Invalid addon index');
        return;
    }

    const addon = addons.value[addonIndex];
    if (!addon || !addon.manifest || !addon.transportUrl) {
        alert('Addon not found or missing transport URL');
        return;
    }

    const addonName = addon.manifest.name || 'Unknown Addon';
    
    // Store original addon details for verification later
    const originalAddonReference = {
        name: addonName,
        transportUrl: addon.transportUrl,
        index: addonIndex
    };
    
    // If this is Cinemeta, use the specialized function
    if (addonName === 'Cinemeta') {
        resetCinemeta();
        return;
    }
    
    console.log(`Resetting ${originalAddonReference.name} to original state...`);
    
    // Set loading state for this specific addon
    isResettingAddon.value = {
        ...isResettingAddon.value,
        [originalAddonReference.index]: true
    };

    // Normalize the transportUrl to a proper manifest.json endpoint
    const manifestUrl = normalizeManifestUrl(addon.transportUrl);

    // Fetch the original manifest
    fetch(manifestUrl)
        .then((response) => {
            if (!response.ok) {
                throw new Error(`Failed to fetch ${originalAddonReference.name} manifest: ${response.status} ${response.statusText}`);
            }
            return response.json();
        })
        .then((originalManifest) => {
            console.log(`Fetched original ${originalAddonReference.name} manifest:`, originalManifest);
            
            // Use validation helper
            const currentAddon = validateAddonResetSafety(
                originalAddonReference, 
                originalManifest, 
                originalAddonReference.index
            );
            
            // Preserve the original addon structure but replace the manifest
            addons.value[originalAddonReference.index] = {
                ...currentAddon,
                manifest: originalManifest
            };

            console.log(`${originalAddonReference.name} addon succesfuly updated`);

            // Auto-sync the updated collection to Stremio
            return syncUserAddons();
        })
        .then(() => {
            console.log(`${originalAddonReference.name} reset and sync completed successfully`);
        })
        .catch((error) => {
            console.error(`Error resetting ${originalAddonReference.name}:`, error);
            alert(`Failed to reset ${originalAddonReference.name}: ${error.message}`);
        })
        .finally(() => {
            // Clear loading state for this specific addon
            const newState = { ...isResettingAddon.value };
            delete newState[originalAddonReference.index];
            isResettingAddon.value = newState;
        });
}

/**
 * Reset Cinemeta addon to its original state by fetching the unmodified manifest
 * from the official source and overwriting the current one, then syncing to Stremio
 */
function resetCinemeta() {
    const key = stremioAuthKey.value;
    if (!key) {
        console.error('No auth key provided');
        alert('Authentication required to reset Cinemeta');
        return;
    }

    // Find the Cinemeta addon in the current collection
    const cinemetaIndex = addons.value.findIndex((addon) => addon && addon.manifest && addon.manifest.name === 'Cinemeta');
    if (cinemetaIndex === -1) {
        alert('Cinemeta addon not found in your collection');
        return;
    }

    console.log('Resetting Cinemeta to original state...');
    isResettingCinemeta.value = true;
    
    // Also set the individual addon resetting state for UI consistency
    isResettingAddon.value = {
        ...isResettingAddon.value,
        [cinemetaIndex]: true
    };

    // Fetch the original Cinemeta manifest from the official source
    fetch('https://v3-cinemeta.strem.io/manifest.json')
        .then((response) => {
            if (!response.ok) {
                throw new Error(`Failed to fetch Cinemeta manifest: ${response.status} ${response.statusText}`);
            }
            return response.json();
        })
        .then((originalManifest) => {
            console.log('Fetched original Cinemeta manifest:', originalManifest);
            
            // Create expected addon reference for validation
            const cinemetaReference = {
                name: 'Cinemeta',
                transportUrl: addons.value[cinemetaIndex].transportUrl
            };
            
            // Use validation helper with Cinemeta-specific checks
            const currentAddon = validateAddonResetSafety(
                cinemetaReference, 
                originalManifest, 
                cinemetaIndex,
                { transportUrlContains: 'cinemeta' }
            );
            
            // Preserve the current addon structure but replace the manifest
            addons.value[cinemetaIndex] = {
                ...currentAddon,
                manifest: originalManifest
            };

            console.log('Cinemeta succesfuly updated');

            // Reset Cinemeta-specific toggles
            shouldRemoveSearchArtifacts.value = false;
            shouldRemoveStandardCatalogs.value = false;
            shouldRemoveMetaResource.value = false;

            // Auto-sync the updated collection to Stremio (now with toggles disabled)
            return syncUserAddons();
        })
        .then(() => {
            // Update Cinemeta-specific patch detection flags after successful sync
            isSearchArtifactsPatched.value = false;
            isStandardCatalogsPatched.value = false;
            isMetaResourcePatched.value = false;
            
            console.log('Cinemeta reset and sync completed successfully');
        })
        .catch((error) => {
            console.error('Error resetting Cinemeta:', error);
            alert(`Failed to reset Cinemeta: ${error.message}`);
        })
        .finally(() => {
            isResettingCinemeta.value = false;
            
            // Clear the individual addon resetting state
            const newState = { ...isResettingAddon.value };
            delete newState[cinemetaIndex];
            isResettingAddon.value = newState;
        });
}
 </script>

<template>
  <section id="configure" class="theme-dark">
    <div class="container">


      <form class="content-grid" @submit.prevent>
        <section class="card auth-card">
          <div class="card-header">
            <div class="step-indicator">1</div>
            <h3>Authenticate</h3>
          </div>
          <div class="card-body">
            <Authentication
              :stremioAPIBase="stremioAPIBase"
              @auth-key="setAuthKey"
              @auth-success="onAuthSuccess"
            />
          </div>
        </section>

        <section class="card" :class="{ disabled: isLocked }" :aria-disabled="isLocked">
          <div class="card-header">
            <div class="step-indicator">2</div>
            <h3>Options</h3>
            <span class="lock-chip" v-if="isLocked">🔒 Authenticate to unlock</span>
          </div>
          <div class="card-body">
            <div class="backup-section">
              <p class="card-desc">Download your current addon configuration before making changes.</p>
              <button
                type="button"
                class="btn accent"
                :disabled="isLocked || addonsCount === 0"
                @click="downloadBackup"
              >
                <span>📥</span>
                <span>Download Backup</span>
              </button>
            </div>
            
            <div class="section-divider"></div>
            
            <div class="option-list">
              <div class="option-row">
                <div class="option-text">
                  <div class="option-title">
                    Remove Cinemeta Search
                    <span v-if="isSearchArtifactsPatched" class="patch-applied-label">Patch applied</span>
                  </div>
                  <div class="option-sub">Stops Cinemeta results from appearing when you search.</div>
                </div>
                <label class="switch">
                  <input type="checkbox" v-model="shouldRemoveSearchArtifacts" :disabled="isLocked || isSearchArtifactsPatched" />
                  <span class="slider"></span>
                </label>
              </div>

              <div class="option-row">
                <div class="option-text">
                  <div class="option-title">
                    Remove Cinemeta Catalogs
                    <span v-if="isStandardCatalogsPatched" class="patch-applied-label">Patch applied</span>
                  </div>
                  <div class="option-sub">Removes Popular, New, and Featured catalogs from your home screen.</div>
                </div>
                <label class="switch">
                  <input type="checkbox" v-model="shouldRemoveStandardCatalogs" :disabled="isLocked || isStandardCatalogsPatched" />
                  <span class="slider"></span>
                </label>
              </div>

              <div class="option-row">
                <div class="option-text">
                  <div class="option-title">
                    Remove Cinemeta Metadata
                    <span v-if="isMetaResourcePatched" class="patch-applied-label">Patch applied</span>
                  </div>
                  <div class="option-sub">Stops Cinemeta from providing metadata for your movies and series.</div>
                </div>
                <label class="switch">
                  <input type="checkbox" v-model="shouldRemoveMetaResource" :disabled="isLocked || isMetaResourcePatched" />
                  <span class="slider"></span>
                </label>
              </div>
            </div>
            
            <div class="section-divider"></div>
            
            <div class="reset-section">
              <p class="card-desc">Reset Cinemeta to its original state, removing all applied patches and restoring default functionality.</p>
              <button
                type="button"
                class="btn primary reset-btn"
                :disabled="isLocked || isResettingCinemeta || addonsCount === 0"
                :aria-busy="isResettingCinemeta"
                @click="resetCinemeta"
              >
                <span class="spinner" v-if="isResettingCinemeta"></span>
                <span>⟲</span>
                <span>Reset Cinemeta</span>
              </button>
            </div>
          </div>
        </section>

        <section class="card" :class="{ disabled: isLocked }" :aria-disabled="isLocked">
          <div class="card-header">
            <div class="step-indicator">3</div>
            <h3>Sync Addons</h3>
            <span class="lock-chip" v-if="isLocked">🔒 Authenticate to unlock</span>
          </div>
          <div class="card-body">
            <p class="card-desc">Apply selected options and sync to Stremio.</p>
            <button
              type="button"
              class="btn primary"
              :disabled="isLocked || isSyncing"
              :aria-busy="isSyncing"
              @click="syncUserAddons"
            >
              <span class="spinner" v-if="isSyncing"></span>
              <span>Sync to Stremio</span>
            </button>
          </div>
        </section>

        <section class="card" :class="{ disabled: isLocked }" :aria-disabled="isLocked">
          <div class="card-header">
            <div class="step-indicator">⚙️</div>
            <h3>Restore</h3>
            <span class="lock-chip" v-if="isLocked">🔒 Authenticate to unlock</span>
          </div>
          <div class="card-body">
            <p class="card-desc">Load and restore a previous addon configuration backup.</p>
            
            <div class="restore-controls">
              <button
                type="button"
                class="btn primary"
                :disabled="isLocked"
                @click="loadBackupFile"
              >
                <span>📁</span>
                <span>Load Backup</span>
              </button>
              
              <div class="file-info" v-if="backupFileName">
                <span class="file-label">Loaded:</span>
                <span class="file-name" :class="{ valid: isBackupValid, invalid: !isBackupValid }">
                  {{ backupFileName }}
                </span>
                <span class="file-status" v-if="isBackupValid">✅</span>
                <span class="file-status" v-else>❌</span>
              </div>
            </div>
            
            <button
              type="button"
              class="btn accent restore-btn"
              :disabled="isLocked || !isBackupValid || isRestoring"
              :aria-busy="isRestoring"
              @click="restoreFromBackup"
            >
              <span class="spinner" v-if="isRestoring"></span>
              <span>Restore Configuration</span>
            </button>
          </div>
        </section>

        <section class="card" :class="{ disabled: isLocked }" :aria-disabled="isLocked">
          <div class="card-header">
            <div class="step-indicator">📦</div>
            <h3>Manage Addons</h3>
            <span class="lock-chip" v-if="isLocked">🔒 Authenticate to unlock</span>
          </div>
          <div class="card-body">
            <p class="card-desc">Create and manage your custom addon configurations directly.</p>
            
            <div class="management-controls">
              <div class="addon-stats">
                <span class="stats-label">Total Addons:</span>
                <span class="stats-count">{{ addonsCount }}</span>
              </div>
           <!---   <button
                type="button"
                class="btn primary"
                :disabled="isLocked"
                @click="toggleAddonManagement"
              >
                <span>{{ isManageAddonVisible ? '📝' : '➕' }}</span>
                <span>{{ isManageAddonVisible ? 'Hide Addon Editor' : 'Create New Addon' }}</span>
              </button> !!WIP!!-->
            </div> 
            
            <div class="current-addons-section" v-if="addonsCount > 0">
              <div class="section-divider"></div>
              <h4 class="section-title">Current Addons <span class="drag-hint">💡 Drag to reorder</span></h4>
              <draggable 
                v-model="addons" 
                class="addons-list sortable-list"
                :disabled="isLocked"
                item-key="id"
                handle=".drag-handle"
                animation="150"
                ghost-class="item-ghost"
                chosen-class="item-chosen"
                drag-class="item-dragging"
                :delay="200"
                :delay-on-touch-only="true"
                :touch-start-threshold="10"
                :force-fallback="false"
                scroll-sensitivity="100"
                bubble-scroll="true"
              >
                <template #item="{ element: addon, index }">
                  <AddonItem
                    :key="index"
                    :name="getNestedObjectProperty(addon, 'manifest.name', 'Unknown Addon')"
                    :idx="index"
                    :manifestURL="addon.transportUrl || 'N/A'"
                    :logoURL="getNestedObjectProperty(addon, 'manifest.logo', null)"
                    :isDeletable="addon.manifest?.name !== 'Cinemeta'"
                    :isConfigurable="!!(addon.transportUrl && addon.transportUrl.includes('/manifest.json'))"
                    :isResetable="!!(addon.transportUrl && addon.transportUrl !== 'N/A')"
                    :isResetting="!!isResettingAddon[index]"
                    @delete-addon="removeAddon"
                    @edit-manifest="openEditModal"
                    @reset-addon="resetAddon"
                  />
                </template>
              </draggable>
            </div>
            
            <div v-if="isManageAddonVisible" class="addon-management-section">
              <div class="section-divider"></div>
              <h4 class="section-title">Addon Configuration</h4>
              <DynamicForm 
                :manifest="newAddonManifest" 
                @update-manifest="saveNewAddon" 
              />
            </div>
          </div>
        </section>
      </form>
    </div>
  </section>

  <div v-if="isEditModalVisible" class="modal theme-dark" @click.self="closeEditModal">
    <div class="modal-content">
      <div class="modal-header">
        <h3>Edit manifest</h3>
        <button class="btn icon ghost" @click="closeEditModal" aria-label="Close modal">×</button>
      </div>
      <div class="modal-body">
        <DynamicForm :manifest="currentManifest" @update-manifest="saveManifestEdit" />
      </div>
    </div>
  </div>
</template>

<style scoped>
/* VS Code dark-inspired theme tokens */
#configure.theme-dark {
  --bg: #1e1e1e;
  --panel: #252526;
  --panel-2: #2d2d30;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
  --accent: #007acc;
  --accent-2: #0e639c;
  --focus: #3794ff;
  --shadow: rgba(0, 0, 0, 0.35);
}

#configure.theme-dark { color: var(--text); padding: 8px 16px; }

.container { max-width: 960px; margin: 0 auto; }

.page-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 12px;
}

.title-wrap .title {
  margin: 0;
  font-size: 22px;
  font-weight: 600;
  letter-spacing: 0.2px;
}

.subtitle {
  margin: 6px 0 0 0;
  color: var(--muted);
  font-size: 13px;
}

.badge {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 6px 10px;
  background: var(--panel-2);
  border: 1px solid var(--border);
  border-radius: 999px;
  font-size: 12px;
  color: var(--text);
}

.content-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 18px;
  width: 100%;
}

/* Cards */
.card {
  width: 100%;
  box-sizing: border-box;
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: 10px;
  box-shadow: 0 6px 18px var(--shadow);
  overflow: hidden;
  transition: transform 120ms ease, box-shadow 120ms ease, border-color 120ms ease;
}
.card:hover {
  transform: translateY(-1px);
  box-shadow: 0 10px 24px var(--shadow);
  border-color: #4b4b4b;
}
.card-header {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 16px;
  border-bottom: 1px solid var(--border);
  background: linear-gradient(180deg, var(--panel), rgba(0, 0, 0, 0));
  min-height: 56px;
}
.card-header h3 {
  margin: 0;
  font-size: 16px;
  font-weight: 600;
}
.lock-chip {
  margin-left: auto;
  font-size: 12px;
  color: var(--muted);
  background: rgba(255,255,255,0.04);
  border: 1px solid var(--border);
  padding: 4px 8px;
  border-radius: 999px;
  white-space: nowrap;
  flex-shrink: 0;
}
.step-indicator {
  width: 28px;
  height: 28px;
  border-radius: 6px;
  background: linear-gradient(180deg, var(--accent), var(--accent-2));
  display: inline-flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-size: 13px;
  font-weight: 700;
  box-shadow: 0 2px 8px rgba(0, 122, 204, 0.35);
}
.card-body { padding: 16px; }
.card.disabled {
  position: relative;
}
.card.disabled::after {
  content: "";
  position: absolute;
  inset: 0;
  background: linear-gradient(180deg, rgba(0,0,0,0.25), rgba(0,0,0,0.35));
  pointer-events: none;
}
.card.disabled .card-body,
.card.disabled .card-header {
  filter: grayscale(0.2) opacity(0.75);
}
.card.disabled .switch .slider {
  background: #2f2f2f;
}
.card.disabled .switch .slider:before {
  background: #a9a9a9;
}
.card-desc {
  color: var(--muted);
  font-size: 13px;
  margin: 0 0 12px 0;
}

/* Vertical stack at all widths; fine-tune spacing on larger screens */
@media (min-width: 720px) {
  #configure.theme-dark { padding: 28px; }
  .content-grid {
    grid-template-columns: 1fr;
    gap: 20px;
  }
  .content-grid > .card { grid-column: auto; }
}
@media (min-width: 1024px) {
  #configure.theme-dark { padding: 32px; }
  .content-grid { gap: 22px; }
}

/* Backup section */
.backup-section {
  margin-bottom: 16px;
}

.backup-section .card-desc {
  margin-bottom: 14px;
}

/* Reset section */
.reset-section {
  margin-top: 16px;
}

.reset-section .card-desc {
  margin-bottom: 14px;
}

.reset-section .reset-btn {
  width: 100%;
  justify-content: center;
}

.section-divider {
  height: 1px;
  background: var(--border);
  margin: 20px 0;
}

/* Restore section */
.restore-controls {
  display: flex;
  align-items: center;
  gap: 16px;
  margin-bottom: 16px;
  flex-wrap: wrap;
}

.file-info {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  background: var(--panel-2);
  border: 1px solid var(--border);
  border-radius: 6px;
  font-size: 13px;
  min-width: 0; /* Allow text to wrap */
}

.file-label {
  color: var(--muted);
  font-weight: 500;
}

.file-name {
  color: var(--text);
  font-family: monospace;
  font-size: 12px;
  max-width: 200px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.file-name.valid {
  color: #4ade80; /* Green for valid */
}

.file-name.invalid {
  color: #f87171; /* Red for invalid */
}

.file-status {
  font-size: 14px;
}

.restore-btn {
  width: 100%;
  justify-content: center;
}

@media (max-width: 640px) {
  .restore-controls {
    flex-direction: column;
    align-items: stretch;
  }
  
  .file-info {
    order: 2;
  }
}

/* Addon management section */
.management-controls {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  margin-bottom: 16px;
  flex-wrap: wrap;
}

.addon-stats {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 12px;
  background: var(--panel-2);
  border: 1px solid var(--border);
  border-radius: 6px;
  font-size: 13px;
}

.stats-label {
  color: var(--muted);
  font-weight: 500;
}

.stats-count {
  color: var(--accent);
  font-weight: 700;
  font-size: 14px;
}

.current-addons-section {
  margin-top: 16px;
}

.addons-list {
  max-height: 400px;
  overflow-y: auto;
  margin-top: 8px;
}

.addon-management-section {
  margin-top: 16px;
}

.section-title {
  margin: 16px 0 12px 0;
  color: var(--text);
  font-size: 15px;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 12px;
}

.drag-hint {
  font-size: 12px;
  color: var(--muted);
  font-weight: 500;
  background: var(--panel-2);
  padding: 4px 8px;
  border-radius: 4px;
  border: 1px solid var(--border);
}

@media (max-width: 640px) {
  .management-controls {
    flex-direction: column;
    align-items: stretch;
  }
  
  .addon-stats {
    order: 1;
    justify-content: center;
  }
}

/* Options list */
.option-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.option-row {
  display: grid;
  grid-template-columns: 1fr auto;
  align-items: center;
  gap: 14px;
  padding: 14px 16px;
  background: var(--panel-2);
  border: 1px solid var(--border);
  border-radius: 8px;
}
.option-title {
  font-size: 14.5px;
  font-weight: 600;
  display: flex;
  align-items: center;
  gap: 8px;
}

.patch-applied-label {
  display: inline-flex;
  align-items: center;
  padding: 2px 5px;
  background: rgba(74, 222, 128, 0.2);
  color: #4ade80;
  border: 1px solid rgba(74, 222, 128, 0.3);
  border-radius: 3px;
  font-size: 9px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.2px;
  white-space: nowrap;
  flex-shrink: 0;
  line-height: 1;
}

/* Mobile optimizations for patch labels */
@media (max-width: 768px) {
  .option-title {
    flex-direction: column;
    align-items: flex-start;
    gap: 4px;
  }
  
  .patch-applied-label {
    font-size: 10px;
    padding: 2px 6px;
    border-radius: 4px;
    letter-spacing: 0.1px;
    align-self: flex-start;
  }
}

@media (max-width: 480px) {
  .patch-applied-label {
    font-size: 10px;
    padding: 3px 6px;
    font-weight: 700;
    background: rgba(74, 222, 128, 0.25);
    border: 1px solid rgba(74, 222, 128, 0.4);
  }
}
.option-sub {
  color: var(--muted);
  font-size: 12.5px;
  margin-top: 4px;
}

/* Toggle switch */
.switch {
  position: relative;
  display: inline-flex;
  align-items: center;
  width: 48px;
  height: 28px;
}
.switch input {
  opacity: 0;
  width: 0;
  height: 0;
}
.slider {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: #3a3a3a;
  border: 1px solid var(--border);
  transition: background 120ms ease, border-color 120ms ease;
  border-radius: 999px;
}
.slider:before {
  position: absolute;
  content: "";
  height: 22px;
  width: 22px;
  left: 3px;
  top: 50%;
  transform: translateY(-50%);
  background-color: #cfcfcf;
  border-radius: 50%;
  box-shadow: 0 2px 6px rgba(0,0,0,0.35);
  transition: transform 120ms ease, background 120ms ease;
}
.switch input:focus + .slider {
  outline: 2px solid var(--focus);
  outline-offset: 2px;
}
.switch input:checked + .slider {
  background: linear-gradient(180deg, var(--accent), var(--accent-2));
  border-color: transparent;
}
.switch input:checked + .slider:before {
  transform: translate(18px, -50%);
  background: white;
}

/* Disabled inputs visual consistency */
.switch input:disabled + .slider {
  background: #2f2f2f;
  border-color: #3b3b3b;
}
.switch input:disabled + .slider:before {
  background: #a9a9a9;
}

/* Locked patch switches (applied patches) - show as enabled but locked */
.switch input:disabled:checked + .slider {
  background: linear-gradient(180deg, rgba(0, 122, 204, 0.7), rgba(14, 99, 156, 0.7));
  border-color: rgba(0, 122, 204, 0.5);
}
.switch input:disabled:checked + .slider:before {
  transform: translate(18px, -50%);
  background: rgba(255, 255, 255, 0.9);
}

/* Buttons */
.btn {
  display: inline-flex;
  align-items: center;
  gap: 10px;
  padding: 11px 16px;
  border-radius: 8px;
  border: 1px solid var(--border);
  background: var(--panel-2);
  color: var(--text);
  font-size: 14.5px;
  line-height: 1;
  cursor: pointer;
  transition: transform 80ms ease, background 120ms ease, border-color 120ms ease, box-shadow 120ms ease;
}
.btn:hover {
  background: #323234;
  border-color: #4b4b4b;
}
.btn:active {
  transform: translateY(1px);
}
.btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}
.btn.primary {
  background: linear-gradient(180deg, #3a3a3a, #2f2f2f);
  border-color: #4a4a4a;
}
.btn.accent {
  background: linear-gradient(180deg, var(--accent), var(--accent-2));
  border-color: transparent;
  color: white;
}
.btn.icon {
  padding: 8px 12px;
  border-radius: 6px;
}
.btn.ghost {
  background: transparent;
  border-color: var(--border);
}

/* Spinner */
.spinner {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  border: 2px solid rgba(255, 255, 255, 0.25);
  border-top-color: rgba(255, 255, 255, 0.95);
  animation: spin 800ms linear infinite;
}
@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

/* Modal */
.modal {
  position: fixed;
  inset: 0;
  z-index: 1000;
  background: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  padding: 20px;
  overflow: auto;
  backdrop-filter: blur(2px);
}
.modal.theme-dark {
  --bg: #1e1e1e;
  --panel: #252526;
  --panel-2: #2d2d30;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
  --accent: #007acc;
  --accent-2: #0e639c;
  --focus: #3794ff;
  --shadow: rgba(0, 0, 0, 0.35);
}
.modal-content {
  width: min(900px, 95vw);
  max-width: calc(100vw - 32px);
  max-height: calc(100vh - 40px);
  background: var(--panel);
  border: 1px solid var(--border);
  border-radius: 10px;
  box-shadow: 0 12px 28px var(--shadow);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}
.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 16px;
  padding: 18px 20px;
  border-bottom: 1px solid var(--border);
  background: var(--panel);
  color: var(--text);
  flex-shrink: 0;
}
.modal-header h3 {
  margin: 0;
  font-size: 17px;
  font-weight: 600;
  color: var(--text);
  line-height: 1.3;
}
.modal-header .btn {
  background: var(--panel-2);
  border: 1px solid var(--border);
  color: var(--text);
  min-width: 36px;
  height: 36px;
  font-size: 18px;
  transition: all 120ms ease;
}
.modal-header .btn:hover {
  background: #323234;
  border-color: #4b4b4b;
  transform: scale(1.05);
}
.modal-body {
  padding: 20px;
  overflow: auto;
  background: var(--panel);
  flex: 1;
  /* Ensure the modal body inherits theme variables for nested components */
  --panel: #252526;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
  --accent: #007acc;
  --accent-2: #0e639c;
  --focus: #3794ff;
}

/* Custom scrollbar for modal body */
.modal-body::-webkit-scrollbar {
  width: 8px;
}
.modal-body::-webkit-scrollbar-track {
  background: var(--panel-2);
  border-radius: 4px;
}
.modal-body::-webkit-scrollbar-thumb {
  background: var(--border);
  border-radius: 4px;
}
.modal-body::-webkit-scrollbar-thumb:hover {
  background: #4a4a4a;
}

/* Drag and Drop Styling */
.sortable-list {
  position: relative;
}

.item-ghost {
  opacity: 0.5;
  background: var(--panel-2);
  border: 2px dashed var(--accent);
  border-radius: 10px;
}

.item-chosen {
  transform: scale(1.02);
  box-shadow: 0 8px 20px rgba(0, 122, 204, 0.25);
  border-color: var(--accent);
}

.item-dragging {
  opacity: 0.8;
  transform: rotate(2deg);
  cursor: grabbing;
}

.item-dragging :where(.details, .actions) {
  opacity: 0.7;
}

/* Mobile modal optimizations */
@media (max-width: 768px) {
  .modal {
    padding: 12px;
    align-items: flex-start;
    padding-top: 20px;
  }
  
  .modal-content {
    width: 100%;
    max-width: none;
    max-height: calc(100vh - 32px);
    border-radius: 8px;
  }
  
  .modal-header {
    padding: 16px;
    gap: 12px;
  }
  
  .modal-header h3 {
    font-size: 16px;
  }
  
  .modal-body {
    padding: 16px;
  }
}

@media (max-width: 480px) {
  .modal {
    padding: 8px;
    padding-top: 16px;
  }
  
  .modal-header {
    padding: 14px;
  }
  
  .modal-body {
    padding: 14px;
  }
}

/* Reduce motion */
@media (prefers-reduced-motion: reduce) {
  * {
    transition: none !important;
    animation: none !important;
  }
}
</style>
