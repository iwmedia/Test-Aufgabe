# Testaufgabe Java Entwickler - Minecraft Report System

**Interwebmedia GmbH**  
Hebbelplatz 2, 25336 Elmshorn

---

## 1. Aufgabenstellung

Entwickle ein serverübergreifendes Report-System für Minecraft-Netzwerke. Das System ermöglicht es Spielern, andere Spieler für Fehlverhalten zu melden. Reports werden asynchron verarbeitet, persistent gespeichert und in Echtzeit an alle Server verteilt. Das System muss sowohl Java Edition als auch Bedrock Edition Spieler unterstützen und berücksichtigt einen Multi-Proxy Aufbau.

## 2. Technische Rahmenbedingungen

### 2.1 Projektstruktur
Maven Multi-Module Projekt mit folgenden Modulen:
- **api** → Interfaces, DTOs, Enums
- **common** → Shared Services, Datenbank-Zugriff, Messaging
- **bukkit** → Spigot/Paper Plugin Implementation  
- **bungee** → BungeeCord Proxy Plugin (Login-Reminder)

### 2.2 Technologie-Stack
- **Java 21** (LTS)
- **Maven** als Build-Tool
- **Paper API 1.21.3** (`io.papermc.paper:paper-api:1.21.3-R0.1-SNAPSHOT`)
- **BungeeCord API** (`net.md-5:bungeecord-api:1.21-R0.1-SNAPSHOT`)
- **MongoDB** mit **Morphia 2.x** (`dev.morphia.morphia:morphia-core:2.4.14`)
- **Redis** mit **Jedis 6** (`redis.clients:jedis:6.0.0`)
- **OkHttp** (`com.squareup.okhttp3:okhttp:4.12.0`)
- **Guava** (`com.google.guava:guava:33.0.0-jre`)
- **Google Guice** (`com.google.inject:guice:7.0.0`)
- **Lombok** (`org.projectlombok:lombok:1.18.38`)
- **SmartInvs** (`fr.minuskube.inv:smart-invs:1.2.7`)
- **Floodgate API** (`org.geysermc.floodgate:api:2.2.4-SNAPSHOT`)
- **InfluxDB Client** (`com.influxdb:influxdb-client-java:7.2.0`)
- **JetBrains Annotations** (`org.jetbrains:annotations:26.0.2`)

## 3. Funktionale Anforderungen

### 3.1 Report-Erstellung

**Command:** `/report [spielername]`

**Verhalten:**
- Ohne Parameter: Öffnet GUI mit allen Online-Spielern (Spielerköpfe)
- Mit Parameter (Spielername): Direkt zum Report-Formular für den angegebenen Spieler

**Offline-Spieler Support:**
- Wenn gemeldeter Spieler nicht online ist, UUID-Auflösung über externe API
- **Edition-Differenzierung**: 
  - Floodgate API nutzen um zu prüfen, ob Spielername ein Bedrock-Spieler sein könnte
  - Bedrock-Spieler haben spezielle Prefixe (standardmäßig `.` oder konfigurierbar)
  - Je nach Edition unterschiedliche API-Endpoints bei mc-api.io verwenden
- Verwendung von **mc-api.io** für UUID-Lookup (Java & Bedrock Support)
- API-Dokumentation: https://mc-api.io/docs
- **Wichtig**: Asynchrone HTTP-Requests, kein Blocking im Main Thread
- Fehlerbehandlung bei API-Ausfall oder ungültigem Spielernamen
- Caching von erfolgreichen Lookups zur Performance-Optimierung

**Report-Templates:**
- `CHEATING` - Verdacht auf Cheats/Hacks
- `INSULT` - Beleidigung/Toxisches Verhalten  
- `BUGUSING` - Ausnutzen von Bugs
- `GRIEFING` - Zerstörung von Bauwerken
- `SPAM` - Chat-Spam/Werbung
- `OTHER` - Sonstiges (mit Pflicht-Freitext)

**Plattform-spezifische Implementierung:**
- **Java Edition**: SmartInventory GUI mit Template-Auswahl
- **Bedrock Edition**: Floodgate Forms mit gleicher Funktionalität

**Prozess-Flow:**
1. Spieler wählt zu meldenden Spieler oder gibt Namen ein
2. Bei Offline-Spieler: UUID-Auflösung über mc-api.io (asynchron)
3. Template-Auswahl (bei OTHER: Freitext erforderlich)
4. Asynchrone Speicherung in MongoDB
5. Chat-Logs der letzten 10 Minuten werden automatisch erfasst
6. Veröffentlichung über Redis Pub/Sub (`reports:new`)
7. Discord Webhook-Benachrichtigung
8. InfluxDB: `report_created` Metrik mit Server und Reason
9. Bestätigung an Reporter: "Dein Report wurde aufgenommen."

### 3.2 Chat-Log System (Bukkit-Ebene)

**Implementierung:**
- AsyncChatEvent Listener auf Bukkit-Server
- Guava Cache mit 10-Minuten Time-Based Eviction
- Cache-Key: Spieler UUID
- Cache-Value: Liste von ChatMessage Objekten

**Automatische Erfassung:**
- Bei jedem Report werden die gecachten Nachrichten des gemeldeten Spielers gespeichert
- Server-lokale Erfassung (nur Nachrichten vom aktuellen Server)
- Asynchrone Speicherung als separates ChatLogModel
- Referenzierung im ReportModel

### 3.3 Team-Benachrichtigungen

**Real-Time Notifications:**
- Redis Channel: `reports:new`
- Alle Bukkit-Server subscriben auf den Channel
- Spieler mit Permission `report.admin` erhalten:
```
[REPORT] <gemeldeter> wurde gemeldet von <reporter> (Grund: <template>)
Verwende /reports, um offene Reports zu verwalten.
```

### 3.4 Report-Verwaltung

**GUI-basierte Verwaltung:**
- Hauptmenü über `/reports` (SmartInventory)
- Filteroptionen: Template, Server, Status, Zeitraum
- Sortierung nach Datum, Server, Template

**Verfügbare Aktionen:**
| Aktion | Beschreibung | Command Alternative |
|--------|--------------|-------------------|
| Resolve | Report als berechtigt abschließen | `/report resolve <id>` |
| Reject | Report mit Begründung ablehnen | `/report reject <id> <grund>` |
| Details | Vollständige Report-Informationen | `/report details <id>` |
| Stats | Statistiken anzeigen | `/report stats [spieler]` |

### 3.5 Report-Status Benachrichtigungen (Multi-Proxy Challenge)

**Problem-Beschreibung:**
In einem Multi-Proxy Setup kann der Reporter auf einem anderen Server/Proxy online sein als der Moderator, der den Report bearbeitet. Beispiel:
- Reporter spielt auf Server "Survival" über Proxy-1
- Moderator bearbeitet Report auf Server "Lobby" über Proxy-2
- Ohne spezielle Implementierung würde der Reporter keine Benachrichtigung erhalten

**Technische Herausforderung:**
- Kein zentraler Spieler-Lookup über alle Proxies
- Jede Proxy kennt nur ihre eigenen Online-Spieler
- Direkte Spieler-zu-Spieler Kommunikation über Proxy-Grenzen nicht möglich

**Anforderung:**
Der Reporter soll bei Abschluss seines Reports benachrichtigt werden:
- **RESOLVED**: `[REPORT] Dein Report gegen <spieler> wurde bearbeitet und als berechtigt eingestuft.`
- **REJECTED**: `[REPORT] Dein Report gegen <spieler> wurde geprüft und abgelehnt. Grund: <modNote>`

**Zu berücksichtigende Aspekte:**
- Benachrichtigung muss Reporter erreichen, egal auf welchem Server/Proxy
- Skalierbarkeit bei vielen Proxies
- Vermeidung von Duplicate Messages
- Performance-Impact minimieren
- Thread-Sicherheit
- Offline-Handling: Was passiert wenn Reporter offline geht?
- Persistence: Sollen verpasste Nachrichten beim nächsten Login zugestellt werden?

### 3.6 Admin Login-Reminder (BungeeCord)

Bei Login eines Spielers mit Permission `report.admin`:
- Asynchrone Prüfung offener Reports
- Nachricht: `[REPORT] Du hast aktuell X offene Reports. Nutze /reports.`

## 4. Datenmodelle & Messaging

### 4.1 MongoDB Datenmodelle

**Report Entity** (`reports` Collection):
```java
@Entity("reports")
public class ReportModel {
    @Id 
    private ObjectId id;
    private UUID reporter;
    private UUID reported;
    private String reporterName;
    private String reportedName;
    private String reason;
    private ReportStatus status; // OPEN, RESOLVED, REJECTED
    private UUID handledBy;
    private String handledByName;
    private String modNote;
    @Reference 
    private ChatLogModel chatLog;
    private String server;
    private Instant createdAt;
    private Instant handledAt;
}
```

**ChatLog Entity** (`chatlogs` Collection):
```java
@Entity("chatlogs")
public class ChatLogModel {
    @Id private ObjectId id;
    private UUID playerUuid;
    private String playerName;
    private List<ChatMessage> messages;
    private Instant capturedAt;
}

public class ChatMessage {
    private String message;
    private String server;
    private Instant timestamp;
}
```

**Indizes:**
- reports: status, reported, reporter, createdAt
- chatlogs: playerUuid, linkedReportId

### 4.2 Redis Pub/Sub

**Channel: `reports:new`**
```json
{
  "reportId": "<ObjectId>",
  "reporter": "<ReporterName>",
  "reported": "<ReportedName>",
  "reason": "<Template-Name oder Custom-Text>",
  "server": "<ServerInstance>",
  "timestamp": "<ISO-8601>"
}
```

**Channel: `reports:status_update`**
```json
{
  "reportId": "<ObjectId>",
  "reporterUuid": "<UUID>",
  "reporterName": "<Name>",
  "reportedName": "<Name>",
  "status": "<RESOLVED|REJECTED>",
  "handledBy": "<ModeratorName>",
  "timestamp": "<ISO-8601>"
}
```

## 5. Integrationen

### 5.1 Discord Webhook
- Formatierte Embed-Nachrichten
- Farbcodierung nach Template
- Enthält: Reporter, Reported, Template, Server, Zeitstempel
- Rein informativ (keine Buttons)
- Rate-Limit Handling
- Asynchroner Versand

### 5.2 InfluxDB Metriken
**Zu erfassende Metriken:**
- **Report-Erstellung**: 
  - Measurement: `report_created`
  - Tags: `server`, `reason` (bei Custom-Grund nur "CUSTOM" als Tag-Wert)
  - Field: `count` (immer 1)
- **Report-Bearbeitungszeit**:
  - Measurement: `report_processing_time`
  - Tags: `server`, `status` (RESOLVED/REJECTED)
  - Field: `duration_minutes` (Zeit zwischen Erstellung und Abschluss)

**Setup-Anforderung:**
- Docker-Compose mit vorkonfiguriertem InfluxDB
- Beispiel-Dashboards oder Queries für Review
- Testdaten für Demonstration

### 5.3 Template-System
- Templates als MongoDB Collection
- Dynamisch erweiterbar
- Struktur: id, name (DE/EN), description (DE/EN)
- Hot-Reload fähig

### 5.4 Externe API Integration
- **mc-api.io** für UUID-Auflösung von Offline-Spielern
- Unterstützt Java Edition und Bedrock Edition
- Asynchrone HTTP-Requests (kein Main Thread Blocking)
- Caching von UUID-Lookups zur Performance-Optimierung
- Fallback-Mechanismus bei API-Ausfall

## 6. Nicht-funktionale Anforderungen

### 6.1 Performance & Threading
- **Keine blockierenden Operationen** im Main Thread
- **Keine eigenen Threads/ExecutorServices**
- Ausschließlich BukkitScheduler/BungeeScheduler
- Alle I/O-Operationen (DB, Redis, Discord, InfluxDB) asynchron

### 6.2 Code-Qualität
- Clean Code Prinzipien
- **Conventional Commits** für alle Commits
- Java 21 Features nutzen (Records, Pattern Matching, Text Blocks)
- Dependency Injection mit Guice
- Sinnvolle Package-Struktur
- JavaDoc für öffentliche APIs

### 6.3 Sicherheit & Stabilität
- Input-Validierung
- Permission-Checks
- Fehlerbehandlung für externe Services
- Reconnect-Mechanismen
- Duplicate-Detection für Spam-Schutz

## 7. Bewertungskriterien

1. **Architektur & Modularität** (20%)
   - Saubere Module-Trennung
   - Wiederverwendbare Komponenten
   - Klare Interfaces

2. **Asynchrone Programmierung** (20%)
   - Korrekte Scheduler-Nutzung
   - Thread-Sicherheit
   - Performance-Optimierung

3. **Datenbank & Messaging** (15%)
   - Effiziente Queries
   - Korrekte Indizierung
   - Fehlerbehandlung

4. **GUI & User Experience** (15%)
   - SmartInventory Implementation
   - Floodgate Forms
   - Intuitive Bedienung

5. **Multi-Proxy Support** (15%)
   - Chat-Log Caching
   - Cross-Proxy Notifications
   - Konsistenz über Instanzen

6. **Code-Qualität & Testing** (15%)
   - Clean Code
   - Test Coverage (min. 70% Business Logic)
   - Dokumentation

## 8. Abgabe

### Lieferumfang
- **Git Repository** mit:
  - Funktionierendem Maven Multi-Module Build
  - Saubere Commit-Historie (Conventional Commits)
  - Branch-Strategie
  
- **README.md** mit:
  - Architektur-Übersicht
  - Setup-Anleitung (MongoDB, Redis, InfluxDB)
  - Build & Deployment
  - Konfigurationsbeispiele
  
- **Docker-Compose** mit:
  - MongoDB Setup
  - Redis Instance
  - InfluxDB mit funktionsfähigem Setup
  - Beispiel-Queries für Metriken-Abfrage
  - Optional: Grafana mit vorkonfigurierten Dashboards

- **Dokumentation**:
  - JavaDoc
  - API-Dokumentation
  - Performance-Überlegungen

### Zeitrahmen
- **Deadline:** 10 Tage nach Aufgabenerhalt
- **Review:** Code Review + Funktionstest + Technisches Gespräch

---

**Wichtige Hinweise:**
- Bei Unklarheiten bitte nachfragen
- Fokus auf Produktionsreife und Skalierbarkeit
- Keine eigenen Thread-Pools - ausschließlich Platform-Scheduler
- Alle I/O-Operationen müssen asynchron erfolgen
