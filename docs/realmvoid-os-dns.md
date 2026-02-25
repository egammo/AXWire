# REALMVOID Architecture Analysis
## Logische Architektur und Kommunikationsfluss

**Analysedatum:** 2026-02-22
**Quelle:** `/mnt/nas-egammo/PROJECTS/REALMVOID/`
**Fokus:** Client/Server/Hive/HiveZone/HiveRelay Kommunikation

---

## 1. KERNKONZEPT

### REALMVOID als Game Universe OS

REALMVOID ist **nicht** eine Game-Engine oder Framework, sondern ein **Backend/Infrastruktur-System**, das folgendes verwaltet:

- Mehrere Game-Universen (realm_families) die gleichzeitig laufen
- Separate Welt-Instanzen (vertices) innerhalb jedes Universums
- Session-Routing und Lifecycle-Management
- Network-Kommunikation zwischen Clients und Servern

**Think of it as:** "Ein kleines Betriebssystem für Online/LAN-Spiele" – ohne Grafik, aber mit Welten, Sessions, Protokollen und vernetzten Komponenten.

### Star Topology Architecture

REALMVOID folgt einem **Star Topology** Pattern:

```
                    REALMVOID CORE
                   (Hub/Authority)
                          |
        ___________________+___________________
        |                  |                   |
     Client A         HiveRelay/Zone      Client B
     (Acknex)         (Network Bridge)     (C#)

      Direct           Relay Monitor       Direct
      Connection       & Orchestration     Connection
```

**Kernprinzip:** MABs sprechen NICHT direkt miteinander. Sie kommunizieren nur durch ihren Host und optional mit REALMVOID Core via AXWire.

---

## 2. CORE COMPONENTS & IHRE ROLLEN

### A. CLIENT (GameClient.cs - C# Acknex Client)

**Verantwortlichkeit:** Gameplay-Loop, User-Input, lokale Visualisierung

**Schlüsselmerkmale:**
- Verbindet via **AXWire-Protokoll** (binär, MessagePack-basiert)
- Unterhält zwei Verbindungstypen:
  1. **Direkte Verbindung** zu einem spezifischen Zone/Vertex-Server
  2. **Relay-Verbindung** zu Relay-Server für Zone-Discovery und Transfer-Orchestrierung

**Authentication Flow:**
```
1. Register bei Relay → Erhalte CUUID (Client UUID)
2. Request Zone-Assignment → Erhalte IP:Port des Game-Servers
3. Verbinde direkt zu Zone → Führe Handshake durch (Authentifizierung)
```

**State Management:**
```csharp
_currentZone = "tutorial"           // Aktuell aktive Zone
_previousZone = "tutorial"          // Für Fallback bei Transfer-Failure
_authenticatedCUUID = Guid          // Server-authoritative eindeutige ID
_isConnected = bool                 // Zone-Verbindungsstatus
_relayConnected = bool              // Relay-Verbindungsstatus
```

**Wichtige Methoden:**
```csharp
ConnectToWorldAsync()               // Initialer Relay-Connect + Handshake
RequestZoneAssignment(zoneName)     // Zone-Assignment von Relay anfordern
ConnectToZone(response)             // Direkte Verbindung zu Zone-Server
TransferToZoneWithRecovery(zone)    // Zone-Transfer mit Auto-Recovery
ReturnToPreviousZone()              // Recovery zu vorheriger Zone
PerformHandshakeAsync()             // CUUID-basierte Authentifizierung
```

---

### B. HIVE RELAY (HiveRelay.cs)

**Verantwortlichkeit:** Network Entry Point, Zone-Assignment, Load-Balancing-Orchestrierung

**Schlüsselmerkmale:**
- **Control Plane**, der den Low-Level RelayServer wrappt
- Verwaltet bis zu 20 Vertices (Hive-Kapazität)
- Implementiert **Reserve Pool Management** (R1-R4 Reserve-Vertices)
- Überwacht Kapazität alle 10 Sekunden
- Promoted/Demoted Vertices automatisch basierend auf Last

**Thermal State Machine für Vertices:**
```
Ember (idle) → Igniting (starting) → Flame (active) → Cooling (stopping) → Ember
```

**Capacity Management Logic:**
```
- Wenn irgendein Vertex > 95% Kapazität: Promote nächsten verfügbaren Reserve-Vertex
- Wenn Gesamt-Kapazität < 50% mit >5 aktiven Vertices: Demote least-loaded Vertex
- Verhindert doppelte Vertex-Registrierung
- Thread-safe mit Lock-Objekten
```

**Kernmethoden:**
```csharp
RegisterVertex(IHiveVertex)         // Vertex beim Relay registrieren
UnregisterVertex(string)            // Vertex-Tracking stoppen
PromoteReserveVertex(string)        // Dormanten Vertex aktivieren
DemoteVertex(string)                // Aktive Vertex-Count reduzieren
GetMetrics()                        // Kapazität/State-Info zurückgeben
StartCapacityMonitoring()           // 10-Sekunden-Monitoring-Loop starten
```

**Load Balancing Strategie:**
```csharp
// Alle 10 Sekunden:
foreach (var vertex in _registeredVertices)
{
    if (vertex.CapacityPercent > 95 && _reserveVertices.Any())
    {
        var reserve = _reserveVertices.First();
        PromoteReserveVertex(reserve.VertexId);
    }
}

if (OverallCapacityPercent < 50 && ActiveVertexCount > 5)
{
    var leastLoaded = FindLeastLoadedVertex();
    DemoteVertex(leastLoaded.VertexId);
}
```

---

### C. HIVE ZONE COORDINATOR (HiveZoneCoordinator.cs)

**Verantwortlichkeit:** Verwaltung von 2-3 Game-Server-Vertices innerhalb einer logischen Zone

**Schlüsselmerkmale:**
- Wrappt AGNON-AX ZoneServer
- Verwaltet bis zu 3 HiveVertices pro Zone (MAX_VERTICES_PER_ZONE = 3)
- Aggregiert Metriken von allen Vertices in der Zone
- Findet least-loaded Vertex für Spieler-Assignment

**Zone Topology Beispiel:**
```
HiveZone "Tutorial"
├── HiveVertex "V01" (60 Spieler / 100 max = 60%)
├── HiveVertex "V02" (40 Spieler / 100 max = 40%)  ← Least loaded
└── HiveVertex "V03" (80 Spieler / 100 max = 80%)

Durchschnitts-Kapazität = (60 + 40 + 80) / 300 = 60%
Spieler wird zu V02 assigned (least loaded)
```

**Verantwortlichkeiten:**
```csharp
AddVertex(IHiveVertex)              // Vertex zu Zone hinzufügen
RemoveVertex(string vertexId)       // Aus Zone entfernen
GetLeastLoadedVertex()              // Load-balanced Routing
GetMetrics()                        // Zone-Level-Statistiken
Start/Stop()                        // Lifecycle-Management
```

**Load-Balancing-Algorithmus:**
```csharp
public IHiveVertex GetLeastLoadedVertex()
{
    return _vertices
        .Where(v => v.CurrentState == HiveVertexState.Flame)
        .OrderBy(v => v.CapacityPercent)
        .FirstOrDefault();
}
```

---

### D. HIVE VERTEX (HiveVertex.cs)

**Verantwortlichkeit:** Wrappt eine einzelne Game-Server-Instanz mit State-Management

**Schlüsselmerkmale:**
- **Wrapper Pattern** (Composition, nicht Inheritance) um AGNON-AX GameServer
- Implementiert Thermal State Machine (Ember → Igniting → Flame → Cooling → Ember)
- Erzwingt Hysterese-Constraints (Mindestzeit im State)
- Trackt aktuelle Spieler und Kapazitäts-Prozentsatz
- Reportet Metriken bei jedem Update-Zyklus

**State Machine Transitions:**
```csharp
// Valide Transitions
Ember → Igniting (nur von Ember)
Igniting → Flame (nur von Igniting)
Flame → Cooling (nur von Flame)
Cooling → Ember (nur von Cooling)
Cooling → Flame (kann zurück transitionieren wenn Kapazität restored)

// Hysterese-Enforcement (5-Minuten-Minimums verhindern Flapping)
if (time_in_state < 5_minutes)
{
    return; // Verhindere Transition
}
```

**Exponierte Metriken:**
```csharp
VertexId: string
CurrentState: HiveVertexState
CurrentPlayers: int
MaxPlayers: int
CapacityPercent: float          // (CurrentPlayers / MaxPlayers) * 100
CPULoad: float
MemoryUsageMB: float
StateEnteredAt: DateTime
LastUpdated: DateTime
```

**Lifecycle:**
```csharp
Start()     → Transition zu Igniting → Flame
Stop()      → Transition zu Cooling → Ember
Update()    → Metrics refresh, State validation
```

---

## 3. KOMMUNIKATIONSFLUSS: CLIENT ZWISCHEN 4 SERVERN WECHSELN

### Initial Setup (4 Server)
```
Server A: Tutorial         (zone: "tutorial", port 27001)
Server B: Town             (zone: "town", port 27002)
Server C: Dungeon          (zone: "dungeon", port 27003)
Server D: Forest           (zone: "forest", port 27004)

+ Relay Server (port 27000) - Route Orchestrator
```

---

### PHASE 1: Connection & Initial Zone Assignment

#### Step 1: Client → Relay Handshake
```
Client.ConnectToWorldAsync()
├─ Erstelle Relay-Verbindung zu 127.0.0.1:27000
├─ Sende HandshakeRequest (neuer Spieler, cuuid=null)
└─ Warte auf HandshakeResponse
    ├─ Server assigned CUUID (z.B. {12345678-1234-...})
    └─ Client cached CUUID lokal für zukünftige Logins
```

**Message-Struktur (HandshakeRequest):**
```csharp
{
    Version: "1.0",
    ClientType: "Acknex",
    CUUID: null,  // Null bei neuem Spieler
    Timestamp: DateTime.UtcNow
}
```

**Response:**
```csharp
{
    Success: true,
    CUUID: {12345678-1234-5678-1234-567812345678},
    WelcomeMessage: "Welcome to REALMVOID",
    ServerVersion: "0.2"
}
```

#### Step 2: Client → Relay Zone Assignment Request
```
Client.RequestZoneAssignment("tutorial")
├─ Sende ZoneAssignmentRequest zu Relay
├─ Relay queried Registry für verfügbaren Vertex
├─ Registry/VERTEXD stellt sicher dass Vertex läuft
└─ Relay antwortet mit:
    {
      AssignedZone: "tutorial",
      ZoneIP: "127.0.0.1",
      ZonePort: 27001,
      Success: true
    }
```

**Relay-Interner Prozess:**
```csharp
1. Lookup zone "tutorial" in registry
2. Find all vertices in state "Flame" for this zone
3. Select least-loaded vertex (HiveZoneCoordinator.GetLeastLoadedVertex())
4. Return IP:Port of selected vertex
5. Register session mapping: CUUID → Vertex
```

#### Step 3: Client → Server A (Tutorial) Direkte Verbindung
```
Client.ConnectToZone(ZoneAssignmentResponse)
├─ _network.Connect("127.0.0.1", 27001)
├─ Warte 1000ms für Verbindungsstabilisierung
├─ PerformHandshakeAsync()
│  ├─ Sende HandshakeRequest mit CUUID
│  └─ Server validiert und antwortet mit:
│     {
│       Success: true,
│       SessionPlayerId: 1,
│       CUUID: {12345678-...},
│       WelcomeMessage: "Welcome to Tutorial"
│     }
└─ Setze CurrentZone = "tutorial"
   Update ClientDatabase.SaveLastZone("tutorial")
```

**State nach Phase 1:**
```
Client:
  _currentZone = "tutorial"
  _previousZone = "tutorial"
  _relayConnected = true
  _isConnected = true (zu Server A)
  _authenticatedCUUID = {12345678-...}
  _zoneIPCache["tutorial"] = "127.0.0.1:27001"

Relay (In-Memory Registry):
  Session S1234 → Server A Vertex (agnon_tutorial#1)

Server A:
  Player#1 authentifiziert und aktiv
```

---

### PHASE 2: Client wechselt von Tutorial → Town

**Der Spieler initiiert:** `client.TransferToZoneWithRecovery("town", "portal")`

#### Step 1: Save Previous Zone (KRITISCH - Task 009 Fix)
```csharp
if (_transferRetryCount == 0)  // Nur beim ersten Versuch!
{
    _previousZone = _currentZone;  // Speichere BEVOR Zone wechselt!
    // Jetzt _previousZone = "tutorial"
}
```

**Warum KRITISCH?**
- Wenn Transfer fehlschlägt, muss Client zur vorherigen Zone zurückkehren
- Wenn _previousZone nicht gespeichert wird, kann Recovery nicht funktionieren
- Bug in frühen Versionen: _previousZone wurde nach Failed Transfer überschrieben

#### Step 2: Request Transfer vom Relay
```
Client.TransferToZone("town")
├─ Validiere CUUID ist nicht leer
├─ Sende ZoneTransferRequest zu Relay:
│  {
│    PlayerId: 1,
│    FromZone: "tutorial",
│    ToZone: "town",
│    Reason: "portal",
│    CUUID: {12345678-...}  // AXNetwork encrypted dies
│  }
├─ Warte auf ZoneTransferResponse (max 10 sec timeout)
└─ Relay verarbeitet:
   ├─ Lookup current session location (Server A, Vertex agnon_tutorial#1)
   ├─ Query registry für town vertex
   │  ├─ Check ob Server B existiert und Kapazität hat
   │  ├─ Falls nicht: VERTEXD startet neue Instanz
   │  └─ Confirm endpoint: 127.0.0.1:27002
   ├─ Move session von Server A → Server B im Registry
   └─ Return ZoneTransferResponse:
      {
        PlayerId: 1,
        Approved: true,
        TargetIP: "127.0.0.1",
        TargetPort: 27002,
        TransferToken: "xyz123"  // Validation Token
      }
```

**Relay-Interner Transfer-Prozess:**
```csharp
// 1. Validierung
if (!_sessions.ContainsKey(cuuid))
    return new ZoneTransferResponse { Approved: false, Reason: "Session not found" };

// 2. Source Vertex Lookup
var currentVertex = _sessions[cuuid].CurrentVertex;

// 3. Target Vertex Lookup
var targetZone = _zoneCoordinators["town"];
var targetVertex = targetZone.GetLeastLoadedVertex();

if (targetVertex == null || targetVertex.CapacityPercent > 95)
{
    // Kein verfügbarer Vertex, versuche Reserve zu promoten
    PromoteReserveVertex("R1");
    targetVertex = targetZone.GetLeastLoadedVertex();
}

// 4. Session Migration
_sessions[cuuid].CurrentVertex = targetVertex;
_sessions[cuuid].PreviousVertex = currentVertex;

// 5. Response
return new ZoneTransferResponse
{
    Approved: true,
    TargetIP = targetVertex.IP,
    TargetPort = targetVertex.Port,
    TransferToken = GenerateToken(cuuid, targetVertex.VertexId)
};
```

#### Step 3: Disconnect von Old Server & Connect zu New Server
```csharp
// Disconnect von Server A
_network.Disconnect();
await Task.Delay(500);

// Cache die neue Zone-Adresse
_zoneIPCache["town"] = "127.0.0.1";
_zonePortCache["town"] = 27002;

// Connect zu Server B (Town)
bool connected = await Task.Run(() =>
    _network.Connect("127.0.0.1", 27002, "")
);

// Warte für Verbindung
_currentZone = "town";
await Task.Delay(1000);

// Authentifiziere mit neuem Server
bool authenticated = await PerformHandshakeAsync();
  └─ Sende HandshakeRequest (returning player, cuuid={12345...})
  └─ Server B validiert und antwortet
```

**Server B Authentication Process:**
```csharp
// Server B empfängt HandshakeRequest mit CUUID
OnHandshakeRequest(HandshakeRequest req)
{
    // 1. CUUID Validation
    if (req.CUUID == null || req.CUUID == Guid.Empty)
        return HandshakeResponse { Success: false };

    // 2. Check mit Relay ob Transfer approved
    var session = _relay.GetSession(req.CUUID);
    if (session.CurrentVertex != this.VertexId)
        return HandshakeResponse { Success: false, Reason: "Not assigned to this vertex" };

    // 3. Load Player Data
    var playerData = _database.LoadPlayer(req.CUUID);

    // 4. Create Session
    var playerId = CreateSession(req.CUUID, playerData);

    // 5. Response
    return new HandshakeResponse
    {
        Success: true,
        SessionPlayerId: playerId,
        CUUID: req.CUUID,
        WelcomeMessage: $"Welcome back to {_zoneName}"
    };
}
```

#### Step 4: Success & Update Database
```csharp
if (authenticated)
{
    Console.WriteLine("Zone transfer successful! Now in: town");
    _clientDb.SaveLastZone("town");  // Update DB
    _transferRetryCount = 0;  // Reset für nächsten Transfer
    return true;
}
```

**State nach Phase 2:**
```
Client:
  _currentZone = "town"
  _previousZone = "tutorial"  // Preserved für Recovery!
  _relayConnected = true
  _isConnected = true (zu Server B)
  _zoneIPCache["tutorial"] = "127.0.0.1:27001"
  _zoneIPCache["town"] = "127.0.0.1:27002"

Relay (Registry):
  Session S1234 → Server B Vertex (agnon_town#1)

Server A:
  Player#1 removed von Session-List

Server B:
  Player#1 added zu Session-List, authenticated
```

---

### PHASE 3: Fast Transfer von Town → Dungeon (mit Cache)

Spieler: `client.TransferToZoneWithRecovery("dungeon", "speedrun")`

#### Step 1-2: (Gleich wie oben)
```
Save: _previousZone = "town"
Request: Sende ZoneTransferRequest("dungeon") zu Relay
Response: Relay returned Server C endpoint (127.0.0.1:27003)
```

#### Step 3: Disconnect & Reconnect
```
_network.Disconnect();  // Von Server B
await _network.Connect("127.0.0.1", 27003);  // Zu Server C
await PerformHandshakeAsync();  // Authenticate
_currentZone = "dungeon";
_zoneIPCache["dungeon"] = "127.0.0.1:27003";
```

**Performance-Optimierung:**
- Cache-Hit: IP:Port aus _zoneIPCache geladen (kein DNS-Lookup)
- Direkte Verbindung (kein Relay-Overhead für Gameplay-Daten)
- Handshake dauert <100ms (nur CUUID-Validation)

---

### PHASE 4: Transfer Failed - Recovery Mechanism

Spieler: `client.TransferToZoneWithRecovery("forest", "explore")`

**Szenario: Server D (Forest) ist offline**

```
Step 1: Save _previousZone = "dungeon"

Step 2: Send ZoneTransferRequest("forest") zu Relay
Response: Transfer DENIED (Server D unreachable)

Step 3: HandleTransferFailure()
├─ _transferRetryCount = 1
├─ Warte 2000ms
└─ Retry TransferToZone("forest")
   └─ Still failed... repeat bis MAX_TRANSFER_RETRIES = 3

Step 4: Alle Retries exhausted!
├─ HandleTransferFailure() triggert ReturnToPreviousZone()
├─ Versuche reconnect zu _previousZone = "dungeon"
│  ├─ Check _zoneIPCache["dungeon"] → Found: 127.0.0.1:27003
│  └─ Connect zu cached Server C
├─ Authenticate auf Server C
└─ Client zurück in "dungeon" sicher!

KRITISCH: Spieler ist NICHT disconnected, nur reverted zu safe zone
_transferRetryCount is reset = 0
Ready für nächsten Transfer-Versuch
```

**Code-Details Recovery:**
```csharp
private async Task<bool> ReturnToPreviousZone()
{
    Console.WriteLine($"Attempting to return to previous zone: {_previousZone}");

    // 1. Check Cache
    if (!_zoneIPCache.ContainsKey(_previousZone))
    {
        Console.WriteLine("ERROR: No cached IP for previous zone!");
        return false; // Fallback: Manual reconnect required
    }

    // 2. Direct Connection (KEIN Relay-Request!)
    var ip = _zoneIPCache[_previousZone];
    var port = _zonePortCache[_previousZone];

    bool connected = await Task.Run(() => _network.Connect(ip, port, ""));
    if (!connected)
    {
        Console.WriteLine("ERROR: Could not reconnect to previous zone!");
        return false;
    }

    // 3. Authenticate
    await Task.Delay(1000);
    bool authenticated = await PerformHandshakeAsync();

    if (authenticated)
    {
        // 4. Update State
        _currentZone = _previousZone;
        Console.WriteLine($"Successfully returned to {_previousZone}");
        return true;
    }

    return false;
}
```

**State nach Failed Transfer + Recovery:**
```
Client:
  _currentZone = "dungeon"  (recovered zu previous zone)
  _previousZone = "dungeon"  (updated nach successful recovery)
  _isConnected = true
  _transferRetryCount = 0  (reset)
  Status: "Safely returned to previous zone"

User sieht: "Transfer to forest failed, returning to dungeon"
           (NO client crash, NO disconnection)
```

**Warum dieser Ansatz funktioniert:**
1. **Cache prevents Relay dependency**: Wenn Relay überlastet ist, cached IP funktioniert noch
2. **Graceful degradation**: Spieler verliert nicht Verbindung, nur Transfer failed
3. **State preservation**: _previousZone ist immer valid (gespeichert bevor Transfer)
4. **User Experience**: Keine Disconnection, nur "Transfer failed, zurück zu sicherer Zone"

---

## 4. SCHLÜSSELARCHITEKTUR-KONZEPTE

### A. CUUID (Client UUID)

**Was es ist:**
- Server-authoritative eindeutige Identifier für einen Spieler
- Generiert von Relay beim ersten Login
- Persistiert lokal beim Client
- Verwendet für Account-Recovery und Session-Validation

**Flow:**
```
Neuer Spieler Login:
1. Client → Relay: "Neuer Spieler, kein CUUID yet"
2. Relay assigned: CUUID = {12345678-...}
3. Client speichert zu Datei: cuuid.dat
4. Relay returned: "Welcome, CUUID = {12345678-...}"
5. Client stores: _authenticatedCUUID = {12345678-...}
6. Zukünftige Logins: Include CUUID in Handshake für Verification
```

**Persistent Storage:**
```csharp
// Client-Side (cuuid.dat)
{12345678-1234-5678-1234-567812345678}

// Server-Side (PlayerDatabase)
CUUID → PlayerData
{
    CUUID: {12345678-...},
    Username: "Player1",
    Level: 42,
    LastZone: "tutorial",
    Inventory: [...],
    Position: (100, 200, 50)
}
```

**Security Implications:**
- CUUID ist encrypted in AXWire-Messages (flag: encrypted = true)
- Server validiert CUUID gegen bekannte Sessions
- Verhindert Session-Hijacking (nur korrekter CUUID kann reconnecten)

---

### B. Session Lifecycle

```
CREATION (Relay assigned):
  client CUUID → relay creates session_id S1234

ROUTING (Registry manages):
  Session S1234 → Vertex agnon_tutorial#1

TRANSFER (Relay orchestrates):
  Session S1234 moves: Vertex#1 → Vertex#2

TERMINATION (Vertex closes):
  Player disconnects → session_id removed
```

**Session Object Structure:**
```csharp
class Session
{
    Guid CUUID;                  // Client-Identifier
    string SessionId;            // "S1234"
    IHiveVertex CurrentVertex;   // Aktueller Vertex
    IHiveVertex PreviousVertex;  // Für Rollback
    DateTime CreatedAt;
    DateTime LastActivity;
    Dictionary<string, object> Metadata;
}
```

**Session State Transitions:**
```
Created → Active → Transferring → Active → Idle → Terminated
   ↓                                ↓               ↓
 Timeout                        Timeout         Cleanup
   ↓                                ↓               ↓
Terminated ← ─────────────────── Terminated ← ─────┘
```

---

### C. Load Balancing Strategy

#### Hive Relay Level:
```csharp
// Alle 10 Sekunden:
private void CapacityMonitoringLoop()
{
    while (_isMonitoring)
    {
        var metrics = GetMetrics();

        // 1. Check Overload
        if (metrics.OverallCapacityPercent > 95)
        {
            var reserve = _reserveVertices.FirstOrDefault(v => v.CurrentState == HiveVertexState.Ember);
            if (reserve != null)
            {
                Console.WriteLine($"Capacity critical ({metrics.OverallCapacityPercent}%), promoting {reserve.VertexId}");
                PromoteReserveVertex(reserve.VertexId);
            }
        }

        // 2. Check Underutilization
        if (metrics.OverallCapacityPercent < 50 && metrics.ActiveVertexCount > 5)
        {
            var leastLoaded = _registeredVertices
                .Where(v => v.CurrentState == HiveVertexState.Flame)
                .OrderBy(v => v.CapacityPercent)
                .FirstOrDefault();

            if (leastLoaded != null && leastLoaded.CapacityPercent < 30)
            {
                Console.WriteLine($"Capacity low ({metrics.OverallCapacityPercent}%), demoting {leastLoaded.VertexId}");
                DemoteVertex(leastLoaded.VertexId);
            }
        }

        Thread.Sleep(10000); // 10 seconds
    }
}
```

#### Zone Coordinator Level:
```csharp
// Wenn Spieler bei Zone ankommt:
public IHiveVertex AssignPlayerToVertex()
{
    // 1. Finde alle Flame-state Vertices in Zone
    var availableVertices = _vertices
        .Where(v => v.CurrentState == HiveVertexState.Flame)
        .ToList();

    if (!availableVertices.Any())
        return null; // Keine verfügbaren Vertices

    // 2. Select Vertex mit NIEDRIGSTER Kapazität
    var leastLoaded = availableVertices
        .OrderBy(v => v.CapacityPercent)
        .First();

    return leastLoaded;
}
```

**Beispiel mit 3 Servern:**
```
Server A: 25 players / 100 = 25% capacity  ← LEAST LOADED
Server B: 60 players / 100 = 60% capacity
Server C: 80 players / 100 = 80% capacity

Neuer Spieler assigned zu Server A
```

**Load Distribution Over Time:**
```
T0:  A=25%, B=60%, C=80%  → Player zu A  → A=26%, B=60%, C=80%
T1:  A=26%, B=60%, C=80%  → Player zu A  → A=27%, B=60%, C=80%
...
T10: A=58%, B=60%, C=80%  → Player zu A  → A=59%, B=60%, C=80%
T11: A=59%, B=60%, C=80%  → Player zu A  → A=60%, B=60%, C=80%
T12: A=60%, B=60%, C=80%  → Player zu A  → A=61%, B=60%, C=80% (A least loaded)

Result: Natural load balancing ohne explizite Player-Migration
```

---

### D. Cache-Based Recovery

**Warum Caching kritisch ist:**

```csharp
Dictionary<string, string> _zoneIPCache;     // "tutorial" → "127.0.0.1"
Dictionary<string, int> _zonePortCache;      // "tutorial" → 27001

// Wenn Transfer failed:
// 1. Don't ask relay (relay might auch unstable sein)
// 2. Use CACHED zone address von successful past connection
// 3. Direct reconnect zu known-good IP:port
// 4. Recovery ohne waiting für relay response
```

**Dies ist der Codex Fix (Task 006Z-fix3):** Verhindert cascading failures wenn Relay überlastet ist.

**Cache Management:**
```csharp
private void UpdateZoneCache(string zoneName, string ip, int port)
{
    _zoneIPCache[zoneName] = ip;
    _zonePortCache[zoneName] = port;

    // Optional: Persist zu Disk für Next Session
    SaveCacheToDisk();
}

private void SaveCacheToDisk()
{
    var cacheData = new
    {
        Zones = _zoneIPCache.Select(kvp => new
        {
            Zone = kvp.Key,
            IP = kvp.Value,
            Port = _zonePortCache[kvp.Key]
        })
    };

    File.WriteAllText("zone_cache.json", JsonSerializer.Serialize(cacheData));
}
```

**Cache Invalidation:**
```csharp
// Nach 24 Stunden: Re-query Relay für fresh IP
if (DateTime.UtcNow - _lastCacheUpdate > TimeSpan.FromHours(24))
{
    RefreshCacheFromRelay();
}
```

---

## 5. MESSAGE FLOW ARCHITECTURE

### AXWire Protocol Structure

```csharp
// Header (24 bytes)
struct AXWireHeader
{
    uint32 version;              // Protocol version (0x0001)
    uint32 category;             // System, Gameplay, Chat
    uint32 msg_type;             // Specific message type
    uint32 flags;                // encrypted, compressed, reliable
    uint64 session_id;           // Identifies session
    uint32 sequence;             // Ordering & reliability
}

// Payload (variable length)
byte[] payload;  // MessagePack serialized object
```

**Message Categories:**
```csharp
enum MessageCategory : uint32
{
    System = 0x01,      // Handshake, ZoneAssignment, Transfer
    Gameplay = 0x02,    // Position, Action, State
    Chat = 0x03,        // Chat messages
    Telemetry = 0x04    // Metrics, Performance
}
```

**Message Types (System Category):**
```csharp
enum SystemMessageType : uint32
{
    HandshakeRequest = 0x0101,
    HandshakeResponse = 0x0102,
    ZoneAssignmentRequest = 0x0103,
    ZoneAssignmentResponse = 0x0104,
    ZoneTransferRequest = 0x0105,
    ZoneTransferResponse = 0x0106,
    ZoneTransferApproved = 0x0107,
    IdentityChallenge = 0x0108,
    AuthParams = 0x0109,
    AuthError = 0x010A
}
```

**Flags:**
```csharp
enum MessageFlags : uint32
{
    None = 0x00,
    Encrypted = 0x01,      // Payload ist encrypted
    Compressed = 0x02,     // Payload ist compressed (zlib)
    Reliable = 0x04,       // Requires ACK
    Ordered = 0x08         // Muss in Reihenfolge delivered werden
}
```

---

### Message Type Routing (Client ↔ Relay)

**Von Relay zu Client:**
```csharp
switch (header.msg_type)
{
    case SystemMessageType.HandshakeResponse:
        HandleRelayHandshakeResponse(payload);
        break;

    case SystemMessageType.ZoneAssignmentResponse:
        HandleZoneAssignmentResponse(payload);
        break;

    case SystemMessageType.ZoneTransferResponse:
    case SystemMessageType.ZoneTransferApproved:  // Newer format, converted
        HandleZoneTransferResponse(payload);
        break;

    case SystemMessageType.IdentityChallenge:  // v1.2 auth
        OnIdentityChallenge(payload);
        break;

    case SystemMessageType.AuthParams:  // v1.2 recovery
        OnAuthParams(payload);
        break;

    case SystemMessageType.AuthError:  // Error feedback
        OnAuthError(payload);
        break;
}
```

**Von Client zu Relay:**
```csharp
// 1. Handshake
SendMessage(new AXWireMessage
{
    Category = MessageCategory.System,
    Type = SystemMessageType.HandshakeRequest,
    Flags = MessageFlags.Encrypted | MessageFlags.Reliable,
    Payload = MessagePackSerializer.Serialize(new HandshakeRequest
    {
        Version = "1.0",
        ClientType = "Acknex",
        CUUID = _authenticatedCUUID
    })
});

// 2. Zone Assignment
SendMessage(new AXWireMessage
{
    Category = MessageCategory.System,
    Type = SystemMessageType.ZoneAssignmentRequest,
    Flags = MessageFlags.Encrypted | MessageFlags.Reliable,
    Payload = MessagePackSerializer.Serialize(new ZoneAssignmentRequest
    {
        ZoneName = "tutorial",
        CUUID = _authenticatedCUUID
    })
});

// 3. Zone Transfer
SendMessage(new AXWireMessage
{
    Category = MessageCategory.System,
    Type = SystemMessageType.ZoneTransferRequest,
    Flags = MessageFlags.Encrypted | MessageFlags.Reliable,
    Payload = MessagePackSerializer.Serialize(new ZoneTransferRequest
    {
        FromZone = _currentZone,
        ToZone = "town",
        Reason = "portal",
        CUUID = _authenticatedCUUID
    })
});
```

---

### Reliability & Ordering

**Reliable Messages (mit ACK):**
```csharp
// Sender
SendReliableMessage(msg)
{
    msg.Sequence = _nextSequence++;
    _pendingAcks[msg.Sequence] = msg;
    SendMessage(msg);

    // Start timeout timer
    StartAckTimer(msg.Sequence, timeout: 5000ms);
}

// Receiver
OnMessageReceived(msg)
{
    if (msg.Flags & MessageFlags.Reliable)
    {
        // Send ACK
        SendAck(msg.Sequence);
    }

    ProcessMessage(msg);
}

// Sender ACK Handler
OnAckReceived(sequence)
{
    _pendingAcks.Remove(sequence);
    CancelAckTimer(sequence);
}

// Timeout Handler
OnAckTimeout(sequence)
{
    if (_retryCount[sequence] < MAX_RETRIES)
    {
        // Resend
        var msg = _pendingAcks[sequence];
        SendMessage(msg);
        _retryCount[sequence]++;
        StartAckTimer(sequence, timeout: 5000ms);
    }
    else
    {
        // Give up
        OnMessageFailed(sequence);
    }
}
```

**Ordered Messages:**
```csharp
// Receiver
Dictionary<uint32, AXWireMessage> _outOfOrderBuffer;
uint32 _expectedSequence = 0;

OnMessageReceived(msg)
{
    if (!(msg.Flags & MessageFlags.Ordered))
    {
        // Non-ordered, process immediately
        ProcessMessage(msg);
        return;
    }

    // Ordered message
    if (msg.Sequence == _expectedSequence)
    {
        // In order, process
        ProcessMessage(msg);
        _expectedSequence++;

        // Check if buffered messages can now be processed
        while (_outOfOrderBuffer.ContainsKey(_expectedSequence))
        {
            ProcessMessage(_outOfOrderBuffer[_expectedSequence]);
            _outOfOrderBuffer.Remove(_expectedSequence);
            _expectedSequence++;
        }
    }
    else if (msg.Sequence > _expectedSequence)
    {
        // Out of order, buffer
        _outOfOrderBuffer[msg.Sequence] = msg;
    }
    // else: Old duplicate, ignore
}
```

---

## 6. STATE MANAGEMENT & SYNCHRONIZATION

### Client State Machine

```
┌─────────────────┐
│  Disconnected   │
└────────┬────────┘
         │
    Connect relay
         ↓
┌─────────────────────┐
│  Relay Connected    │  (kann jetzt Zone Assignments machen)
│  (Unauthenticated)  │
└────────┬────────────┘
         │
    Request zone assignment
    & connect zu zone
         ↓
┌──────────────────────┐
│ Zone Connected       │
│ (Authenticating)     │
└────────┬─────────────┘
         │
    Handshake complete
         ↓
┌──────────────────────┐
│ Zone Connected       │  ← Kann spielen, kann Zones transferieren
│ (Authenticated)      │
└────────┬─────────────┘
         │
    Initiate transfer
         ↓
┌──────────────────────┐
│ Transferring         │  (zwischen Zones)
│ (Zone intermediate)  │
└────────┬─────────────┘
         │
    New zone connected
    & authenticated
         ↓
┌──────────────────────┐
│ Zone Connected       │  (ready für next transfer)
│ (Authenticated)      │
└─────────────────────┘
```

**Code Implementation:**
```csharp
enum ClientState
{
    Disconnected,
    RelayConnected,
    ZoneConnecting,
    ZoneAuthenticating,
    ZoneConnected,
    Transferring
}

ClientState _currentState = ClientState.Disconnected;

// State Transitions
void TransitionToState(ClientState newState)
{
    Console.WriteLine($"State transition: {_currentState} → {newState}");

    // Exit actions
    switch (_currentState)
    {
        case ClientState.ZoneConnected:
            SavePlayerState();
            break;
    }

    _currentState = newState;

    // Entry actions
    switch (newState)
    {
        case ClientState.RelayConnected:
            StartRelayMonitoring();
            break;

        case ClientState.ZoneConnected:
            LoadPlayerState();
            StartGameplayLoop();
            break;
    }
}
```

---

### Relay Monitoring

```csharp
Timer _relayMonitorTimer;
int _reconnectAttempts = 0;
const int MAX_RECONNECT_ATTEMPTS = 10;

// Timer läuft alle 5 Sekunden:
void MonitorRelayConnection()
{
    _relayMonitorTimer = new Timer(5000);
    _relayMonitorTimer.Elapsed += (s, e) =>
    {
        // Check ob Relay noch connected
        if (!_relayNetwork.IsConnected)
        {
            Console.WriteLine("Relay connection lost, attempting reconnect...");

            if (_reconnectAttempts < MAX_RECONNECT_ATTEMPTS)
            {
                bool reconnected = _relayNetwork.Reconnect();
                _reconnectAttempts++;

                if (reconnected)
                {
                    Console.WriteLine("Relay reconnected successfully");
                    _reconnectAttempts = 0;
                }
            }
            else
            {
                Console.WriteLine("Max reconnect attempts reached, giving up");
                _relayMonitorTimer.Stop();
                OnRelayPermanentlyLost();
            }
        }
        else
        {
            // Connected, reset attempt counter
            _reconnectAttempts = 0;
        }
    };

    _relayMonitorTimer.Start();
}
```

---

### Server-Side State Synchronization

**Vertex → Relay Metrics Reporting:**
```csharp
// HiveVertex sendet Metrics alle 1 Sekunde
Timer _metricsTimer;

void StartMetricsReporting()
{
    _metricsTimer = new Timer(1000);
    _metricsTimer.Elapsed += (s, e) =>
    {
        var metrics = new VertexMetrics
        {
            VertexId = this.VertexId,
            CurrentPlayers = this.CurrentPlayers,
            MaxPlayers = this.MaxPlayers,
            CapacityPercent = this.CapacityPercent,
            CPULoad = GetCPULoad(),
            MemoryUsageMB = GetMemoryUsage(),
            CurrentState = this.CurrentState,
            Timestamp = DateTime.UtcNow
        };

        _relay.UpdateVertexMetrics(metrics);
    };

    _metricsTimer.Start();
}
```

**Relay Aggregation:**
```csharp
// HiveRelay aggregated Metrics von allen Vertices
Dictionary<string, VertexMetrics> _vertexMetrics;

void UpdateVertexMetrics(VertexMetrics metrics)
{
    _vertexMetrics[metrics.VertexId] = metrics;

    // Trigger capacity monitoring wenn update received
    if (ShouldTriggerCapacityCheck(metrics))
    {
        CheckCapacityAndAdjust();
    }
}

bool ShouldTriggerCapacityCheck(VertexMetrics metrics)
{
    // Immediate check bei critical thresholds
    return metrics.CapacityPercent > 95 || metrics.CapacityPercent < 10;
}
```

---

## 7. ZUSAMMENFASSUNG: DAS GROSSE BILD

```
┌─────────────────────────────────────────────────────────┐
│              REALMVOID UNIVERSE                          │
│                                                          │
│  realm_family = "agnon"                                 │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  realm_id = "tutorial"                           │   │
│  │  ├─ vertex#1 (V01): 60 players, 100 max, Flame   │   │
│  │  ├─ vertex#2 (V02): 40 players, 100 max, Flame   │   │
│  │  └─ vertex#3 (V03): 80 players, 100 max, Flame   │   │
│  │  = HiveZone(Tutorial) managed by ZoneCoordinator │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  realm_id = "town"                               │   │
│  │  ├─ vertex#1 (V04): 45 players, 100 max, Flame   │   │
│  │  └─ vertex#2 (V05): 75 players, 100 max, Flame   │   │
│  │  = HiveZone(Town) managed by ZoneCoordinator     │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  ... (Forest, Dungeon zones folgen gleichem Pattern)    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  RESERVE VERTICES (R1-R4): Ember state            │   │
│  │  Promoted zu Flame wenn capacity > 95%           │   │
│  │  Demoted zurück zu Ember wenn capacity < 50%     │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │  HiveRelay (Control Plane)                        │   │
│  │  ├─ Capacity monitoring (10-sec cycle)            │   │
│  │  ├─ Load balancing orchestration                  │   │
│  │  └─ Zone assignment routing                       │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
└─────────────────────────────────────────────────────────┘

                          ↑
                      AXWire
                      Protocol
                          ↓

┌─────────────────────────────────────────────────────────┐
│  CLIENT SIDE (Acknex/C# Game)                           │
│                                                          │
│  GameClient                                             │
│  ├─ Relay connection: zone assignment + monitoring      │
│  ├─ Zone connection: gameplay                           │
│  ├─ State tracking:                                     │
│  │  ├─ _currentZone                                    │
│  │  ├─ _previousZone (für recovery)                    │
│  │  ├─ _authenticatedCUUID                            │
│  │  └─ Connection caches (IP:port per zone)            │
│  └─ Transfer orchestration:                             │
│     ├─ Save previous zone                              │
│     ├─ Request transfer von relay                      │
│     ├─ Disconnect old zone                              │
│     ├─ Connect new zone                                 │
│     └─ On failure: return zu previous zone              │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 8. KEY TAKEAWAYS

### Der Logische Flow (Was passiert wenn Client zwischen 4 Servern wechselt)

#### 1. Initial State: Client in Server A (Tutorial)
```
Zone: "tutorial"
IP: 127.0.0.1:27001
RelayConnected: true
Authenticated: true
```

#### 2. Spieler Initiiert Transfer: "Gehe zu Town"
```
1. Save _previousZone = "tutorial"
2. Send ZoneTransferRequest("town") zu Relay
3. Relay findet Server B mit Capacity und confirmed Endpoint
```

#### 3. Disconnect & Reconnect:
```
1. Disconnect von Server A
2. Connect zu Server B (127.0.0.1:27002)
3. Perform Handshake Authentication
4. Update _currentZone = "town"
```

#### 4. Cache Updates:
```
_zoneIPCache["tutorial"] = "127.0.0.1:27001"  (für recovery)
_zoneIPCache["town"] = "127.0.0.1:27002"
```

#### 5. On Failure:
```
1. Retry bis zu 3 Mal (mit 2-second delays)
2. Falls alle Retries failed: Automatisch return zu _previousZone
3. Use cached IP address (don't ask overloaded relay)
4. Reconnect zu known-good server
5. KRITISCH: Don't crash client, nur revert zone
```

#### 6. Success:
```
1. Update database mit new zone
2. Ready für next transfer
3. _previousZone preserved für recovery on next transfer
```

---

### Die Konzepte Dahinter

#### CUUID: Server-Authoritative Player Identity
- Generiert beim ersten Login
- Persistiert lokal und server-side
- Used für alle zukünftigen Authentifications
- Encrypted in allen Nachrichten

#### Relay: Smart Orchestrator
- NICHT ein direct data proxy (low-latency routes)
- Control plane für Zone Discovery
- Load Balancing Decisions
- Session Registry Management

#### State Machine: Vertices haben Thermal States
- Ember → Igniting → Flame → Cooling → Ember
- Hysterese (5 Minuten minimum) verhindert Flapping
- Automatic Provisioning/Deprovisioning based auf demand

#### Composition Pattern:
- Relay wraps RelayServer
- Vertex wraps GameServer
- Zone wraps ZoneServer
- Separation of Concerns: Metrics & Orchestration vs. Game Logic

#### Cache-Based Recovery:
- Verhindert cascading failures wenn relay busy
- Direct reconnection zu known-good IP:Port
- No dependency auf relay für recovery
- Fast recovery (< 1 second)

#### Load Balancing:
- Least-loaded vertex innerhalb zone bekommt neue players
- Automatic capacity monitoring (10 second cycles)
- Reserve pool (R1-R4) für auto-scaling

#### Auto-Scaling:
- Reserve vertices (R1-R4) automatisch promoted bei demand
- Demoted bei low utilization
- Graceful shutdown (Cooling state)
- No player disruption während scaling

---

### Die Architektur ermöglicht:

✅ **Indie-scale multiplayer:** 100-500 CCU pro zone
✅ **Clear separation of concerns:** Control vs. Data Plane
✅ **Visible metrics:** Echtzeit capacity monitoring
✅ **Graceful degradation:** Failures isoliert, no cascading
✅ **Low latency:** Direct client-to-server connections
✅ **Auto-scaling:** Automatic provisioning based auf demand
✅ **State preservation:** No player data loss bei transfers
✅ **Recovery mechanisms:** Automatic fallback zu safe zones

---

## 9. IMPLEMENTATION CHECKLIST FÜR NEUIMPLEMENTIERUNG

### Phase 1: Core Protocol (AXWire)
```
[ ] Message Header Structure (24 bytes)
[ ] MessagePack Serialization
[ ] Encryption Layer (Flags: Encrypted)
[ ] Compression Layer (Flags: Compressed)
[ ] Reliable Messaging (ACK/Retry)
[ ] Ordered Messaging (Sequence Numbers)
[ ] Message Categories (System, Gameplay, Chat)
[ ] Message Type Routing
```

### Phase 2: Client Components
```
[ ] Network Layer (AXNetwork)
[ ] Dual Connection Management (Relay + Zone)
[ ] State Machine (Disconnected → Connected → Authenticated)
[ ] CUUID Management (Generation, Persistence, Validation)
[ ] Zone Cache (IP:Port storage)
[ ] Handshake Flow
[ ] Zone Assignment Request/Response
[ ] Zone Transfer Request/Response
[ ] Transfer Retry Logic (MAX_RETRIES = 3)
[ ] Recovery Mechanism (ReturnToPreviousZone)
[ ] Relay Monitoring Timer (5 sec)
```

### Phase 3: Server Components (Relay)
```
[ ] RelayServer Wrapper (HiveRelay)
[ ] Session Registry (CUUID → Vertex Mapping)
[ ] Vertex Registration
[ ] Zone Assignment Logic (GetLeastLoadedVertex)
[ ] Transfer Orchestration
[ ] Capacity Monitoring (10 sec timer)
[ ] Reserve Pool Management (R1-R4)
[ ] Promote/Demote Logic
[ ] Metrics Aggregation
```

### Phase 4: Server Components (Zone/Vertex)
```
[ ] GameServer Wrapper (HiveVertex)
[ ] Thermal State Machine (Ember/Igniting/Flame/Cooling)
[ ] Hysterese Logic (5 min minimum)
[ ] Player Session Management
[ ] Metrics Reporting (1 sec timer)
[ ] Zone Coordinator (HiveZoneCoordinator)
[ ] Multi-Vertex Zone Support (MAX 3 per zone)
[ ] Least-Loaded Selection Algorithm
```

### Phase 5: Testing Scenarios
```
[ ] Basic Connection Flow (Relay → Zone)
[ ] CUUID Generation & Persistence
[ ] Zone Assignment (Single Server)
[ ] Zone Transfer (2 Servers)
[ ] Zone Transfer with Cache (3+ Servers)
[ ] Transfer Failure & Recovery
[ ] Relay Reconnection
[ ] Capacity Monitoring & Auto-Scaling
[ ] Reserve Promotion (>95% capacity)
[ ] Vertex Demotion (<50% capacity)
[ ] Concurrent Transfers (100+ clients)
[ ] Network Partition Scenarios
```

---

## 10. REFERENZEN

### Absolute File Paths (Source Material)

#### Dokumentation:
- `/mnt/nas-egammo/PROJECTS/REALMVOID/docs/REALMVOID_Manifest_v0_2.md`
- `/mnt/nas-egammo/PROJECTS/REALMVOID/docs/REALMVOID_Renderer_MAB_ABI_v0_2.md`
- `/mnt/nas-egammo/PROJECTS/REALMVOID/docs/REALMVOID_Spezifikation_v0_4.md`
- `/mnt/nas-egammo/PROJECTS/REALMVOID/docs/REALMVOID_Konzept_Overview_v0_1.md`
- `/mnt/nas-egammo/PROJECTS/REALMVOID/docs/REALMVOID_Realm_Vertex_Model_v0_1.md`
- `/mnt/nas-egammo/PROJECTS/REALMVOID/docs/REALMVOID_MAB_Bundles_v0_1.md`

#### Source Code:
- `/mnt/nas-egammo/PROJECTS/REALMVOID/Legacy_Reference/AGNON-HIVE/src/AGNON.Hive/Core/HiveRelay.cs`
- `/mnt/nas-egammo/PROJECTS/REALMVOID/Legacy_Reference/AGNON-HIVE/src/AGNON.Hive/Core/HiveVertex.cs`
- `/mnt/nas-egammo/PROJECTS/REALMVOID/Legacy_Reference/AGNON-HIVE/src/AGNON.Hive/Core/HiveZoneCoordinator.cs`
- `/mnt/nas-egammo/PROJECTS/REALMVOID/Legacy_Reference/AGNON-HIVE/src/AGNON.Client/Network/GameClient.cs`
- `/mnt/nas-egammo/PROJECTS/REALMVOID/Legacy_Reference/AGNON-HIVE/src/AGNON.Client/Network/ServerConnection.cs`

---

## 11. GLOSSAR

| Begriff | Definition |
|---------|-----------|
| **REALMVOID** | Backend-System für Multiplayer-Game-Management |
| **CUUID** | Client UUID - Server-authoritative Player Identifier |
| **AXWire** | Binary Protocol (MessagePack-based) für Client-Server Communication |
| **Relay** | Control Plane Server für Zone Discovery & Transfer Orchestration |
| **Zone** | Logische Game-Welt (z.B. "tutorial", "town") |
| **Vertex** | Physische Server-Instanz die eine Zone hostet |
| **Hive** | Cluster von bis zu 20 Vertices managed by HiveRelay |
| **HiveRelay** | Wrapper um RelayServer mit Capacity Monitoring |
| **HiveVertex** | Wrapper um GameServer mit State Machine |
| **HiveZoneCoordinator** | Manager für 2-3 Vertices innerhalb einer Zone |
| **Thermal State** | Vertex State (Ember/Igniting/Flame/Cooling) |
| **Hysterese** | Minimum time in state (5 min) um Flapping zu verhindern |
| **Reserve Pool** | R1-R4 Vertices in Ember state für Auto-Scaling |
| **Session** | Server-side Mapping: CUUID → Current Vertex |
| **Transfer** | Client move von einer Zone zu anderer |
| **Recovery** | Automatic return zu previous zone bei transfer failure |
| **Cache** | Client-side IP:Port storage für fast reconnection |
| **MAB** | Minimal Autonomous Blockchain (REALMVOID component concept) |
| **ABI** | Application Binary Interface (MAB interaction spec) |

---

## ENDE

**Analysiert durch:** Claude Sonnet 4.5
**Agent ID:** a7e7c90
**Datum:** 2026-02-22
**Version:** 1.0
