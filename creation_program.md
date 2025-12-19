```markdown
# Documentation Technique : Système de Gestion des Programmes Voltalis

## Vue d'ensemble

Cette intégration Home Assistant gère les programmes de chauffage Voltalis en les exposant comme des entités Switch. Les programmes peuvent être de deux types : **USER** (définis par l'utilisateur) ou **DEFAULT** (programmes par défaut du système).

## Architecture

### Structure des fichiers

1. **`aiovoltalis/program.py`** : Modèle de données `VoltalisProgram`
2. **`aiovoltalis/__init__.py`** : API client avec méthodes de récupération et modification
3. **`controller.py`** : Contrôleur Home Assistant qui orchestre les mises à jour
4. **`switch.py`** : Plateforme Switch qui expose les programmes comme entités
5. **`entity.py`** : Classe de base pour les entités programmes

## 1. Modèle de Données

### Enum ProgramType

```python
class ProgramType(Enum):
    DEFAULT = "DEFAULT"  # Programmes par défaut du système
    USER = "USER"        # Programmes définis par l'utilisateur
```

### Classe VoltalisProgram

**Propriétés principales :**
- `id` : Identifiant unique du programme (int)
- `name` : Nom du programme (string)
- `isEnabled` : État activé/désactivé (bool)
- `_program_type` : Type du programme (ProgramType)
- `_program_json` : Données brutes JSON du programme

**Méthodes :**
- `async_update()` : Met à jour uniquement les programmes USER depuis l'API

## 2. Appels API

### URLs de base

```
BASE_URL = "https://api.myvoltalis.com"
PROGRAMMING_PROGRAMS_URL = BASE_URL + "/api/site/__site__/programming/program"
QUICK_SETTINGS_URL = BASE_URL + "/api/site/__site__/quicksettings"
```

**Note :** `__site__` doit être remplacé par l'ID du site récupéré lors de l'authentification.

### 2.1 Récupération initiale des programmes

#### Programmes utilisateur (USER)

**Endpoint :** `GET /api/site/{site_id}/programming/program`

**Méthode :** `async_get_programs()` dans `aiovoltalis/__init__.py`

**Processus :**
1. Appel API GET sur `PROGRAMMING_PROGRAMS_URL`
2. Pour chaque programme JSON reçu :
   - Créer une instance `VoltalisProgram` avec `ProgramType.USER`
   - Stocker dans le dictionnaire `_programs` avec l'ID comme clé

**Code :**
```python
programs_json = await self.async_send_request(
    CONST.PROGRAMMING_PROGRAMS_URL, 
    retry=False, 
    method=CONST.HTTPMethod.GET
)
for program_json in programs_json:
    program = VoltalisProgram(program_json, self, ProgramType.USER)
    self._programs[program.id] = program
```

#### Programmes par défaut (DEFAULT)

**Endpoint :** `GET /api/site/{site_id}/quicksettings`

**Processus :**
1. Appel API GET sur `QUICK_SETTINGS_URL`
2. Pour chaque programme JSON reçu :
   - Créer une instance `VoltalisProgram` avec `ProgramType.DEFAULT`
   - Stocker dans le dictionnaire `_programs` avec l'ID comme clé

**Code :**
```python
programs_json = await self.async_send_request(
    CONST.QUICK_SETTINGS_URL, 
    retry=False, 
    method=CONST.HTTPMethod.GET
)
for program_json in programs_json:
    program = VoltalisProgram(program_json, self, ProgramType.DEFAULT)
    self._programs[program.id] = program
```

**Résultat :** Retourne une liste de tous les programmes (USER + DEFAULT)

### 2.2 Mise à jour des programmes

#### Mise à jour des programmes par défaut (batch)

**Endpoint :** `GET /api/site/{site_id}/quicksettings`

**Méthode :** `async_update_default_programs()`

**Processus :**
1. Appel API GET pour récupérer tous les programmes par défaut
2. Pour chaque programme reçu, mettre à jour le `_program_json` du programme existant

**Code :**
```python
programs_json = await self.async_send_request(
    CONST.QUICK_SETTINGS_URL, 
    retry=False, 
    method=CONST.HTTPMethod.GET
)
for program_json in programs_json:
    self._programs[program_json["id"]]._program_json = program_json
```

#### Mise à jour d'un programme utilisateur (individuel)

**Endpoint :** `GET /api/site/{site_id}/programming/program/{program_id}`

**Méthode :** `async_update_user_program(program_id: int)`

**Processus :**
1. Appel API GET avec l'ID du programme
2. Mettre à jour le `_program_json` du programme correspondant

**Code :**
```python
program_json = await self.async_send_request(
    f"{CONST.PROGRAMMING_PROGRAMS_URL}/{program_id}",
    retry=False,
    method=CONST.HTTPMethod.GET,
)
self._programs[program_id]._program_json = program_json
```

### 2.3 Modification de l'état des programmes

#### Activation/Désactivation d'un programme par défaut

**Endpoint :** `PUT /api/site/{site_id}/quicksettings/{program_id}/enable`

**Méthode :** `async_set_default_program_state(program_id, **kwargs)`

**Body JSON :**
```json
{
    "enabled": true  // ou false
}
```

**Code :**
```python
await self.async_send_request(
    f"{CONST.QUICK_SETTINGS_URL}/{program_id}/enable",
    retry=False,
    method=CONST.HTTPMethod.PUT,
    json={"enabled": state}
)
```

#### Activation/Désactivation d'un programme utilisateur

**Endpoint :** `PUT /api/site/{site_id}/programming/program/{program_id}`

**Méthode :** `async_set_user_program_state(program_id, **kwargs)`

**Body JSON :**
```json
{
    "name": "Nom du programme",
    "enabled": true  // ou false
}
```

**Note :** Pour les programmes USER, il faut inclure le nom dans le body.

**Code :**
```python
await self.async_send_request(
    f"{CONST.PROGRAMMING_PROGRAMS_URL}/{program_id}",
    retry=False,
    method=CONST.HTTPMethod.PUT,
    json={
        "name": program.name,
        "enabled": state
    }
)
```

## 3. Intégration Home Assistant

### 3.1 Initialisation dans le Controller

**Fichier :** `controller.py`

**Étape 1 : Récupération initiale**

Lors de `async_setup_entry()` :
```python
self.programs = await self._voltalis.async_get_programs()
```

**Étape 2 : Création du Coordinator**

Un `DataUpdateCoordinator` est créé pour gérer les mises à jour périodiques :
```python
self.coordinator = DataUpdateCoordinator(
    self._hass,
    _LOGGER,
    name=DOMAIN,
    update_method=self.async_update_data,
    update_interval=timedelta(seconds=SCAN_INTERVAL),
)
```

**Étape 3 : Méthode de mise à jour**

La méthode `async_update_data()` est appelée périodiquement :
```python
async def async_update_data(self):
    """Query the API and return the new state."""
    try:
        # Mise à jour des appliances...
        
        # Mise à jour des programmes
        async with asyncio.timeout(POLLING_TIMEOUT):
            for program in self.programs:
                await program.async_update()  # Met à jour uniquement les USER
            await self._voltalis.async_update_default_programs()  # Met à jour tous les DEFAULT
        
    except VoltalisException as err:
        raise UpdateFailed(err) from err
```

**Étape 4 : Enregistrement des devices**

Chaque programme est enregistré comme un device dans le registre Home Assistant :
```python
def async_register_devices(self, entry):
    device_registry = dr.async_get(self._hass)
    
    for program in self.programs:
        device_registry.async_get_or_create(
            config_entry_id=entry.entry_id,
            identifiers={(DOMAIN, str(program.id))},
            name=program.name.capitalize(),
            entry_type=dr.DeviceEntryType.SERVICE
        )
```

### 3.2 Plateforme Switch

**Fichier :** `switch.py`

**Étape 1 : Setup de la plateforme**

La fonction `async_setup_entry()` est appelée par Home Assistant lors de l'initialisation :
```python
async def async_setup_entry(
    hass: HomeAssistant, 
    entry: ConfigEntry, 
    async_add_entities: AddEntitiesCallback
) -> None:
    """Setup Switch Entities."""
    controller = hass.data[DOMAIN][entry.entry_id][VOLTALIS_CONTROLLER]
    entities = []
    
    # Créer une entité Switch pour chaque programme
    for program in controller.programs:
        entities.append(VoltalisProgram(controller.coordinator, program))
    
    async_add_entities(entities)
```

**Étape 2 : Classe VoltalisProgram (SwitchEntity)**

La classe hérite de `VoltalisEntity` (qui hérite de `CoordinatorEntity`) et `SwitchEntity` :

```python
class VoltalisProgram(VoltalisEntity, SwitchEntity):
    """Voltalis program."""
    
    _attr_has_entity_name = True
    _attr_name = None
    _attr_icon = "mdi:toggle-switch"
    
    def __init__(self, coordinator, program):
        """Initialize the entity."""
        super().setupProgram(coordinator, program)
        self.coordinator = coordinator
```

**Étape 3 : Propriété d'état**

```python
@property
def is_on(self) -> bool:
    """Get Switch State."""
    return self.program.isEnabled
```

**Étape 4 : Méthodes de contrôle**

```python
async def async_turn_on(self, **kwargs) -> None:
    """Set state to ON."""
    await self.async_set_state(True)
    await self.coordinator.async_refresh()

async def async_turn_off(self, **kwargs) -> None:
    """Set state to OFF."""
    await self.async_set_state(False)
    await self.coordinator.async_refresh()
```

**Étape 5 : Logique de modification d'état**

La méthode `async_set_state()` gère différemment les programmes USER et DEFAULT :

```python
async def async_set_state(self, state: bool) -> None:
    """Set the state through the API."""
    if self.program._program_type == ProgramType.USER:
        # Pour les programmes USER : inclure le nom
        curjson = {
            "name": self.program.name,
            "enabled": state
        }
        await self.program.api.async_set_user_program_state(
            json=curjson,
            program_id=self.program.id
        )
    else:
        # Pour les programmes DEFAULT : seulement enabled
        curjson = {
            "enabled": state
        }
        await self.program.api.async_set_default_program_state(
            json=curjson,
            program_id=self.program.id
        )
    
    # Rafraîchir les données après modification
    await self.coordinator.async_refresh()
```

### 3.3 Classe de base VoltalisEntity

**Fichier :** `entity.py`

La méthode `setupProgram()` configure les informations de base de l'entité :

```python
def setupProgram(
    self,
    coordinator: DataUpdateCoordinator,
    program: VoltalisProgram,
) -> None:
    """Initialize the entity."""
    super().__init__(coordinator)  # Initialise CoordinatorEntity
    self.program = program
    self._attr_unique_id = str(program.id)
    self._attr_device_info = DeviceInfo(
        identifiers={(DOMAIN, str(program.id))},
        name=program.name.capitalize(),
        manufacturer='Voltalis',
        model='Heater Program',
    )
```

## 4. Flux complet de fonctionnement

### 4.1 Au démarrage de l'intégration

1. **Authentification** : L'utilisateur se connecte avec email/password
2. **Récupération du site ID** : `async_get_default_site_id()`
3. **Récupération des appliances** : `async_get_appliances()`
4. **Récupération des programmes** : `async_get_programs()`
   - Appel GET sur `/programming/program` → programmes USER
   - Appel GET sur `/quicksettings` → programmes DEFAULT
5. **Création du coordinator** : Configuration des mises à jour périodiques
6. **Enregistrement des devices** : Chaque programme est enregistré
7. **Création des entités Switch** : Une entité par programme

### 4.2 Mise à jour périodique

Le coordinator appelle `async_update_data()` à intervalles réguliers :

1. **Mise à jour des appliances** (hors scope programmes)
2. **Mise à jour des programmes USER** :
   - Pour chaque programme USER : `program.async_update()`
   - Appel GET individuel sur `/programming/program/{id}`
3. **Mise à jour des programmes DEFAULT** :
   - Appel GET batch sur `/quicksettings`
   - Mise à jour de tous les programmes DEFAULT en une fois

### 4.3 Interaction utilisateur (activation/désactivation)

1. **Utilisateur active/désactive un switch** dans Home Assistant
2. **Appel de `async_turn_on()` ou `async_turn_off()`**
3. **Appel de `async_set_state(state)`**
4. **Détection du type de programme** :
   - Si USER → `async_set_user_program_state()`
   - Si DEFAULT → `async_set_default_program_state()`
5. **Appel API PUT** avec le body JSON approprié
6. **Rafraîchissement** : `coordinator.async_refresh()` pour mettre à jour l'état

## 5. Points importants à retenir

### Différences USER vs DEFAULT

1. **Récupération** :
   - USER : Endpoint `/programming/program`
   - DEFAULT : Endpoint `/quicksettings`

2. **Mise à jour** :
   - USER : Appel individuel par programme (`/programming/program/{id}`)
   - DEFAULT : Appel batch pour tous (`/quicksettings`)

3. **Modification d'état** :
   - USER : Endpoint `/programming/program/{id}`, body avec `name` et `enabled`
   - DEFAULT : Endpoint `/quicksettings/{id}/enable`, body avec seulement `enabled`

### Gestion des erreurs

- Les appels API utilisent `retry=False` pour éviter les retentatives automatiques
- Les exceptions sont capturées dans `async_update_data()` et remontées comme `UpdateFailed`
- Le coordinator gère automatiquement les erreurs et réessaie selon sa configuration

### Performance

- Les programmes DEFAULT sont mis à jour en batch (plus efficace)
- Les programmes USER sont mis à jour individuellement (nécessaire car ils peuvent être modifiés par l'utilisateur)
- Le coordinator limite les appels simultanés avec `asyncio.timeout()`

## 6. Structure JSON attendue

### Programme USER (réponse API)

```json
{
    "id": 12345,
    "name": "Mon Programme",
    "enabled": true
}
```

### Programme DEFAULT (réponse API)

```json
{
    "id": 67890,
    "name": "Confort",
    "enabled": false
}
```

### Body pour modification USER

```json
{
    "name": "Mon Programme",
    "enabled": true
}
```

### Body pour modification DEFAULT

```json
{
    "enabled": true
}
```

## 7. Checklist pour réimplémentation

Pour recoder ce système dans une autre intégration, suivre ces étapes :

- [ ] Créer l'enum `ProgramType` (USER/DEFAULT)
- [ ] Créer la classe modèle `Program` avec propriétés id, name, isEnabled
- [ ] Implémenter `async_get_programs()` : récupérer USER et DEFAULT
- [ ] Implémenter `async_update_user_program(id)` : GET individuel
- [ ] Implémenter `async_update_default_programs()` : GET batch
- [ ] Implémenter `async_set_user_program_state(id, json)` : PUT avec name+enabled
- [ ] Implémenter `async_set_default_program_state(id, json)` : PUT avec enabled
- [ ] Initialiser les programmes dans le controller au démarrage
- [ ] Créer le coordinator avec méthode de mise à jour
- [ ] Enregistrer les programmes comme devices
- [ ] Créer la plateforme Switch avec `async_setup_entry()`
- [ ] Implémenter la classe SwitchEntity avec `is_on`, `async_turn_on`, `async_turn_off`
- [ ] Gérer la différence USER/DEFAULT dans `async_set_state()`
- [ ] Rafraîchir le coordinator après chaque modification
```

Ce fichier markdown documente le système de gestion des programmes. Vous pouvez le sauvegarder et l'utiliser pour guider une autre IA dans la réimplémentation.