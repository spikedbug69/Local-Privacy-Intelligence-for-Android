# Local Privacy Intelligence for Android

## Product concept, technical design, learning roadmap and execution plan

**Working title:** Local Privacy Intelligence (LPI)\
**Platform:** Android\
**Core idea:** A lightweight, offline-first Android application that
continuously records privacy-relevant signals about installed apps and
turns them into understandable explanations using a small language model
(SLM) that runs locally.

------------------------------------------------------------------------

## 1. Executive summary

The application helps a user understand, in plain language, what
installed applications appear able to access, what sensitive permissions
they actually use (where Android exposes that information), which
network services they contact, and what those services are commonly
associated with.

The user does **not** need to see packet contents, messages, passwords,
or the exact personal data transmitted. The product should provide a
brief, evidence-based picture:

-   What categories of information an app can access.
-   Which sensitive permissions were observed in use.
-   Which domains or services the app contacted.
-   Whether those destinations are associated with analytics,
    advertising, crash reporting, authentication, content delivery, or
    other purposes.
-   What the observations might mean.
-   What is known, what is inferred, and what cannot be established.

All monitoring records, analysis, reports and SLM inference should
remain on the device. Network access may be used optionally to update a
tracker/domain intelligence database, but it must not be required for
core monitoring or reporting.

### Product principle

**Always-on lightweight monitoring; occasional on-device AI analysis.**

Do not run the language model continuously. Continuous packet/event
processing should be handled by efficient Android services and
deterministic code. Invoke the SLM when a user requests an explanation,
when a meaningful pattern is detected, or during a scheduled report.

------------------------------------------------------------------------

## 2. Problem statement

Privacy tools commonly expose separate pieces of information:

-   Static app analysis identifies permissions and tracker SDKs.
-   Network monitors show destinations and connection metadata.
-   Android system privacy indicators show some recent camera,
    microphone and location usage.
-   Privacy policies describe what developers say they collect.

These sources are useful, but they do not automatically form one
understandable, chronological explanation for an ordinary user.

The proposed application combines available signals into a local privacy
timeline and app-level report. It should avoid claiming that a
particular personal datum was transmitted unless there is direct
evidence for that claim.

### Example user-facing summary

> **Instagram --- last 24 hours**
>
> -   Location permission: access observed twice.
> -   Camera permission: access observed once.
> -   Microphone: permission is available; no access was observed by
>     this monitor.
> -   Network activity: connections to Meta-associated and
>     analytics-associated domains were observed.
> -   Possible purposes: application functionality, analytics,
>     advertising measurement or personalization.
>
> **Interpretation:** The app used location and camera permissions
> during the observation period and contacted services associated with
> Meta and analytics. The available evidence does not prove that
> location coordinates or camera content were transmitted.

This distinction between an observation and an inference is a core
product requirement.

------------------------------------------------------------------------

## 3. Goals and non-goals

### Goals

1.  Monitor supported privacy signals continuously with low overhead.
2.  Attribute network activity to apps where Android APIs and device
    conditions permit.
3.  Maintain a local, searchable history.
4.  Classify known domains and tracker services using a local
    intelligence database.
5.  Explain collected evidence in concise, non-technical language with
    an on-device SLM.
6.  Show uncertainty and evidence provenance for every conclusion.
7.  Work without a cloud backend or account.
8.  Let users pause monitoring, clear records, export reports and
    control retention.

### Non-goals for the first release

-   Decrypting all HTTPS traffic.
-   Reading messages, passwords, photos or other private app content.
-   Bypassing certificate pinning.
-   Rooting the phone or requiring a custom Android ROM.
-   Claiming to know every data field an app collects internally.
-   Guaranteeing that every background event is observable on every
    Android device.
-   Replacing Android's built-in permission controls or a full network
    firewall.

------------------------------------------------------------------------

## 4. High-level architecture

``` text
                    ANDROID DEVICE
                           |
             User consent and monitor controls
                           |
                 Monitoring foreground service
                           |
             +-------------+--------------+
             |                            |
      Permission signals             Network signals
      (supported AppOps,              (VpnService,
       system indicators,              packet metadata,
       package metadata)               DNS where visible)
             |                            |
             +-------------+--------------+
                           |
                   Event normalizer
                           |
                 Local rules/classifiers
                           |
          +----------------+----------------+
          |                |                |
    App inventory    Domain intelligence   Risk rules
    and permissions  and tracker database  and correlation
          |                |                |
          +----------------+----------------+
                           |
                    Room local database
                           |
             Trigger and report scheduler
                           |
                 On-device SLM (optional)
                           |
               Evidence-grounded explanation
                           |
                 Jetpack Compose dashboard
```

### Separation of responsibilities

-   **Android monitoring layer:** obtains signals that Android makes
    available.
-   **Rules and classifiers:** perform deterministic counting, matching,
    correlation and scoring.
-   **Database:** stores observations and source/provenance metadata.
-   **SLM:** explains structured evidence; it does not monitor packets
    or decide facts by itself.
-   **UI:** displays observations, confidence, limitations and user
    controls.

------------------------------------------------------------------------

## 5. Technical feasibility and Android limitations

This project is feasible as a normal Android app, but Android does not
expose unrestricted system-wide visibility to third-party apps. The
implementation must be designed around those restrictions.

### 5.1 App inventory and declared permissions

Use Android's `PackageManager` to enumerate installed packages and
inspect requested permissions, subject to Android package-visibility
rules and the relevant Google Play policy.

This answers: **What has the app declared or requested permission to
access?**

It does not prove the app has actually accessed the resource.

### 5.2 Permission access history

Investigate:

-   `AppOpsManager` and the operations it exposes.
-   Android privacy indicators and Privacy Dashboard behaviour.
-   Runtime permission state and package metadata.
-   Version-specific APIs and OEM differences.

This may provide evidence of some sensitive operations, but a
third-party app cannot assume it can read a complete, unrestricted,
real-time history for every other app. Validate each permission and
Android version on physical devices. Do not present an unavailable event
as proof that access did not occur.

### 5.3 Network monitoring

`VpnService` can create a local VPN interface and receive IP packets
routed through it. A local-only monitor can inspect packet metadata
without forwarding traffic to a remote VPN server.

Potentially observable signals include:

-   Source/destination IP addresses.
-   Protocol and port.
-   Packet and byte counts.
-   Connection timing and frequency.
-   DNS queries when DNS traffic is visible to the monitor.
-   Domain-to-IP associations where they can be reliably established.

Important limitations:

-   HTTPS/TLS payloads remain encrypted.
-   DNS-over-HTTPS, DNS-over-TLS, QUIC, VPNs, private relays, cached DNS
    and other mechanisms can reduce domain visibility.
-   Mapping every connection to an app is not universally
    straightforward. Investigate Android APIs such as
    `ConnectivityManager.getConnectionOwnerUid()` where applicable, plus
    supported per-app VPN routing and UID/package mapping.
-   Android permits only one active VPN service per user/profile. The
    monitor may conflict with another VPN or firewall app.
-   A VPN-based monitor must have a clear user disclosure, consent flow,
    notification and policy-compliant purpose.

Do not intercept, decrypt or modify user traffic in the MVP.

### 5.4 Encryption

TLS encryption does not prevent the product from recording connection
metadata and correlating it with permission events and known tracker
destinations.

The product should report:

-   **Observed:** a permission access event or connection was recorded.
-   **Known association:** a destination appears in the local
    intelligence database as an analytics or advertising service.
-   **Inferred:** the combination may be consistent with analytics,
    personalization or advertising.
-   **Unknown:** the exact payload or personal data transmitted cannot
    be established.

Never turn a temporal correlation into a claim that a specific item of
data was sent.

### 5.5 Background execution

Use a user-started foreground monitoring service for continuous network
monitoring, with a persistent notification and clear pause/stop
controls. Follow the Android version's foreground-service type,
permission and background-start requirements.

Android may still stop or restrict processes under some conditions.
Battery optimization, OEM power management, reboot behaviour and Android
version differences must be tested. Do not promise uninterrupted
monitoring.

Use `WorkManager` for deferrable tasks such as daily summaries, database
cleanup and intelligence-database updates. WorkManager is not a
substitute for a continuous network monitor.

------------------------------------------------------------------------

## 6. Recommended technology stack

  -----------------------------------------------------------------------
  Layer                   Recommendation          Reason
  ----------------------- ----------------------- -----------------------
  Primary language        **Kotlin**              Native Android APIs,
                                                  coroutines and
                                                  first-class Android
                                                  tooling

  UI                      **Jetpack Compose**     Modern native Android
                                                  UI

  App architecture        MVVM or unidirectional  Separates UI, state and
                          data flow               monitoring logic

  Background monitoring   Android Foreground      User-visible continuous
                          Service                 monitoring

  Network visibility      Android `VpnService`    Local packet metadata
                                                  observation

  Permission signals      `AppOpsManager`,        App permissions and
                          PackageManager and      available
                          supported system APIs   access-operation
                                                  signals

  Local persistence       Room over SQLite        Structured, queryable
                                                  event history

  Async/background jobs   WorkManager             Scheduled reports,
                                                  cleanup and database
                                                  updates

  SLM runtime             **LiteRT-LM for         On-device LLM inference
                          Android/Kotlin**        
                          (evaluate current       
                          release and device      
                          support)                

  Alternative model       llama.cpp / MLC LLM,    Alternative deployment
  runtime                 after benchmarking      options

  Domain intelligence     Bundled local           Offline classification
                          tracker/domain lists    

  Testing                 JUnit, AndroidX Test,   Unit and device-level
                          physical Android        validation
                          devices                 

  Version control         Git and GitHub          Source control and
                                                  issue tracking
  -----------------------------------------------------------------------

### Language choices

-   **Kotlin:** use for the Android app, services, database, UI and
    Android API integration.
-   **Python:** optional for offline data preparation, tracker-list
    cleaning, test-data generation, model evaluation and analysis
    scripts. Do not make Python a required runtime component on the
    phone.
-   **C/C++:** not needed initially. Consider it only if profiling
    proves packet parsing or model inference needs native optimization.
-   **SQL:** learn enough to design Room entities, indexes, queries and
    retention operations.

Do not start with Flutter or React Native. The project depends heavily
on Android-specific APIs, so native Kotlin reduces integration friction.

------------------------------------------------------------------------

## 7. SLM design

### 7.1 What the model should do

The model receives a compact, structured evidence summary and generates
a short explanation. It should not receive raw packet streams.

Useful tasks:

-   Turn permission and network observations into plain-language
    summaries.
-   Explain the likely role of a known tracker category.
-   Compare this week's activity with the user's baseline.
-   Explain why an app received a risk indicator.
-   Answer questions using the app's local evidence records.
-   Generate daily or weekly privacy reports.

### 7.2 What the model should not do

-   Continuously inspect packets.
-   Decide whether a domain is malicious without supporting
    intelligence.
-   Invent data categories or collection purposes.
-   Claim that encrypted payloads were inspected.
-   Replace deterministic risk rules with an ungrounded model score.
-   Run in a tight loop or remain active merely to appear always-on.

### 7.3 Candidate models

Start by benchmarking small quantized models rather than committing to
one model based on parameter count alone. Candidate families to evaluate
include:

-   Qwen small models.
-   Gemma small models.
-   SmolLM family.

Check the model's licence, mobile runtime compatibility, supported
quantization, actual memory footprint, latency, output quality and
redistribution terms before selecting it. Model formats are
runtime-specific; a model checkpoint may need conversion before use with
LiteRT-LM.

**Recommended first experiment:** build the app's monitoring and rules
engine first, then compare two small quantized models using the same set
of evidence-grounded prompts. Keep the model optional so the app remains
useful when it is not installed or cannot run on a device.

### 7.4 Triggered inference

Use the SLM only when:

1.  The user opens an app's explanation.
2.  The user requests a privacy report.
3.  A rules-based detector identifies a meaningful new pattern.
4.  A scheduled daily/weekly summary is due and the device is
    idle/charging, if appropriate.

The SLM can be loaded for inference and released afterwards, subject to
runtime behaviour. Avoid assuming that unloading instantly returns all
memory to the operating system.

### 7.5 Evidence-grounded output

Pass structured facts with explicit labels:

``` json
{
  "app": "Example App",
  "period": "last_24_hours",
  "observations": [
    {
      "type": "permission_access",
      "permission": "location",
      "count": 2,
      "source": "supported_android_operation_signal"
    },
    {
      "type": "network_destination",
      "domain": "analytics.example",
      "category": "analytics",
      "classification_source": "local_domain_database"
    }
  ],
  "limitations": [
    "HTTPS payload was not inspected",
    "domain category is based on local intelligence data"
  ]
}
```

The response should use a constrained structure, for example:

-   Observed facts
-   Possible interpretation
-   What cannot be confirmed
-   Evidence references

The UI should allow the user to inspect the observations behind each
AI-generated statement.

------------------------------------------------------------------------

## 8. Local data model

Suggested Room entities:

### `InstalledApp`

-   packageName
-   appName
-   versionName
-   firstSeen
-   lastSeen

### `DeclaredPermission`

-   packageName
-   permissionName
-   granted
-   lastChecked

### `PermissionEvent`

-   packageName
-   operation
-   timestamp
-   observationSource
-   confidence / availability status

### `NetworkEvent`

-   packageName or UID (nullable if attribution is unavailable)
-   destinationIp
-   destinationPort
-   protocol
-   domain (nullable)
-   bytesSent
-   bytesReceived
-   timestamp
-   attributionSource

### `DomainIntelligence`

-   domain or suffix
-   organization
-   category
-   source
-   listVersion
-   lastUpdated

### `DailyAppSummary`

-   packageName
-   date
-   permission counts
-   domain categories
-   connection counts
-   bytes sent/received
-   limitations

### `AnalysisReport`

-   packageName
-   period
-   generatedAt
-   modelId / rulesVersion
-   structured findings
-   evidence event IDs

Use indexes on package name, timestamp, domain and event type. Aggregate
repeated network events to avoid storing every packet indefinitely. Set
a configurable retention period and provide a clear "delete all local
history" action.

------------------------------------------------------------------------

## 9. Privacy classification and risk scoring

Start with transparent rules, not an opaque AI-generated score.

Possible dimensions:

1.  **Permission sensitivity:** what sensitive permissions are declared
    and granted.
2.  **Observed access:** whether a supported access event was recorded.
3.  **Destination classification:** known analytics, advertising, crash
    reporting, authentication or content delivery domains.
4.  **Activity pattern:** frequency, timing and changes compared with
    the app's own baseline.
5.  **Evidence quality:** direct observation, database association or
    inference.

Do not automatically label an app "high risk" just because it has many
permissions or uses analytics. A camera app needs camera access; a
messaging app may contact analytics and crash-reporting services. Risk
must consider purpose, context, frequency and evidence quality.

Show the basis of every score. Keep "privacy exposure" separate from
"maliciousness"; they are not the same thing.

------------------------------------------------------------------------

## 10. Main application screens

### A. Overview dashboard

-   Monitoring status.
-   Apps with the most observed privacy activity.
-   Recent sensitive permission events.
-   Recently contacted tracker/third-party categories.
-   Daily/weekly trend.

### B. Installed apps

For each app: - Declared/granted permissions. - Supported
permission-access history. - Network destinations and categories. - Data
usage and connection frequency. - AI-generated explanation. - Evidence
and limitations.

### C. Privacy timeline

A chronological event stream, for example:

``` text
09:15  Example App — location access observed
09:16  Example App — connection to api.example.com
09:16  Destination classified as first-party service
10:30  Another App — analytics domain contacted
```

Do not imply that events close together prove data transmission.

### D. Ask the privacy assistant

Examples: - "Which apps accessed location today?" - "Which apps
contacted advertising-related domains?" - "What changed compared with
last week?" - "Why is this app marked as having elevated privacy
exposure?"

Answers must be based on local database records and must say when
evidence is unavailable.

### E. Settings and transparency

-   Start/pause monitoring.
-   VPN consent and status.
-   Retention period.
-   Local intelligence database version.
-   SLM enable/disable and model status.
-   Export/delete local records.
-   Explanation of what is and is not monitored.

------------------------------------------------------------------------

## 11. Learning prerequisites

Learn these in roughly this order.

### Android and Kotlin fundamentals

-   Kotlin syntax, classes, interfaces, null safety and collections.
-   Coroutines, Flow and structured concurrency.
-   Android activity/service lifecycle.
-   Intents, permissions and application components.
-   Manifest declarations and runtime permissions.
-   Jetpack Compose and state management.
-   MVVM or unidirectional data flow.

### Android system and privacy APIs

-   PackageManager and package visibility.
-   Runtime permission model.
-   AppOpsManager and its limitations.
-   Privacy Dashboard and privacy indicators.
-   Foreground services and notifications.
-   WorkManager and Doze/battery restrictions.
-   Android version and OEM compatibility.

### Networking

-   OSI and TCP/IP models.
-   IPv4/IPv6, TCP, UDP and ports.
-   DNS resolution and DNS caching.
-   TLS/HTTPS and certificate chains.
-   SNI, encrypted DNS and QUIC basics.
-   VPNs, TUN interfaces and Android VpnService.
-   Packet metadata versus payload inspection.
-   UID-to-package attribution and its limitations.

### Data engineering

-   SQL and relational modelling.
-   SQLite and Room.
-   Indexes, aggregation and retention.
-   Event timestamps and time zones.
-   Data minimization and local encryption where appropriate.

### Privacy and security

-   Android sandbox and UID isolation.
-   Least privilege.
-   Consent and transparent disclosure.
-   Threat modelling.
-   Secure local storage.
-   False positives, evidence quality and confidence.
-   Google Play policies for VPN-based apps and sensitive data.

### Small language models

-   Tokens, context windows and inference.
-   Quantization (for example, 4-bit).
-   RAM, KV cache and model storage.
-   Prompt design and structured output.
-   Hallucination and grounding.
-   On-device inference runtimes.
-   Benchmarking latency, memory and energy.
-   Model licences and distribution constraints.

------------------------------------------------------------------------

## 12. Execution plan

A realistic solo prototype plan is approximately **10--12 weeks at 8--12
focused hours per week**. This is an estimate, not a guarantee. A
polished, Play-Store-ready application will take longer, especially
because Android monitoring behaviour varies by device and OS version.

  -----------------------------------------------------------------------
  Week                    Workstream              Deliverable
  ----------------------- ----------------------- -----------------------
  1                       Kotlin, Android Studio, Basic app shell and
                          Git and Compose setup   navigation

  2                       Package inventory and   Installed-app list with
                          permission metadata     declared/granted
                                                  permissions

  3                       Room database and event Local event storage and
                          model                   history screen

  4                       AppOps/system signal    Demonstration of which
                          research and prototype  permission events are
                                                  actually observable

  5                       VpnService fundamentals User-consented local
                          and packet metadata     VPN monitor prototype

  6                       UID/app attribution,    Per-app network event
                          DNS and domain matching records where
                                                  attribution is
                                                  supported

  7                       Local tracker/domain    Category labels and
                          intelligence and rules  transparent rule-based
                                                  findings

  8                       Timeline and app        Useful non-AI MVP
                          privacy reports         

  9                       SLM runtime integration Local explanation
                          and model evaluation    generated from
                                                  structured evidence

  10                      Triggering, scheduled   On-demand AI analysis
                          reports and resource    with no continuous
                          profiling               inference

  11                      Device testing,         Compatibility and
                          battery/RAM profiling   performance report
                          and false-positive      
                          review                  

  12                      Security review, UX     Demonstrable prototype
                          polish and              and project report
                          documentation           
  -----------------------------------------------------------------------

### Phase gates

**Gate 1 --- End of week 4:** Prove what permission-use information a
normal third-party app can actually observe on target Android versions.
If a signal is unavailable, redesign the UI rather than implying
complete visibility.

**Gate 2 --- End of week 6:** Prove that the network monitor can collect
useful metadata and attribute a meaningful proportion of connections to
apps without breaking connectivity.

**Gate 3 --- End of week 8:** The non-AI product must already provide
useful reports. If it does not, adding an SLM will not fix the
underlying data problem.

**Gate 4 --- End of week 10:** The SLM must produce grounded, concise
explanations and must not invent transmission claims.

**Gate 5 --- End of week 12:** Verify battery use, memory, stability,
data retention, disclosure and user controls on multiple physical
devices.

------------------------------------------------------------------------

## 13. MVP scope

Do not attempt every feature in the first build.

### MVP must have

-   Installed-app inventory.
-   Declared/granted permission view.
-   Only those permission-use events that can be reliably observed.
-   User-started local VpnService monitor.
-   Basic connection metadata and best-effort app attribution.
-   Local tracker/domain classification.
-   Room-based history.
-   App-level privacy summary with explicit limitations.
-   Start/stop controls and persistent notification.

### MVP should have

-   Daily summary.
-   Simple rule-based exposure indicators.
-   Export and delete history.
-   Offline operation after installation and intelligence-list setup.

### Later versions

-   SLM-generated explanations.
-   Natural-language queries over local records.
-   Cross-app third-party correlation.
-   Baseline and anomaly detection.
-   Better domain intelligence updates.
-   Optional advanced diagnostics for rooted/test devices (not required
    for normal users).

------------------------------------------------------------------------

## 14. Testing and success criteria

Test on physical devices, not only the Android emulator. Include at
least two Android versions and, if possible, devices from different
manufacturers.

Measure:

-   Percentage of observed network connections attributed to the correct
    app.
-   Percentage of DNS/domain events successfully classified.
-   Which permission events are visible, per Android version and device.
-   False-positive rate for tracker and risk classifications.
-   Battery drain compared with monitoring disabled.
-   Idle and peak RAM usage.
-   CPU time and SLM inference latency.
-   Database growth per day.
-   Whether monitoring survives screen-off, app navigation, network
    changes and device reboot under supported configurations.
-   Whether another VPN disables or conflicts with the monitor.
-   Whether the app remains useful when the SLM is disabled.

Do not set a fixed RAM or battery promise before measurement. Establish
a baseline, profile on representative phones and optimize based on
results.

------------------------------------------------------------------------

## 15. Risks and mitigations

  -----------------------------------------------------------------------
  Risk                                Mitigation
  ----------------------------------- -----------------------------------
  Android hides or limits permission  Validate APIs early; show coverage
  history                             limitations

  VPN conflicts with another VPN      Detect conflict and explain that
                                      only one VPN can be active per
                                      profile

  Domain attribution is incomplete    Store confidence/source;
                                      distinguish IP from confirmed
                                      domain

  Encrypted DNS hides queries         Use other available metadata;
                                      report unknown rather than guessing

  SLM hallucinations                  Feed structured evidence; require
                                      evidence references and constrained
                                      output

  Battery drain                       Batch events, aggregate traffic,
                                      avoid continuous inference

  Large model footprint               Make SLM optional; benchmark
                                      quantized candidates

  OEM background restrictions         Test multiple devices and explain
                                      persistence limits

  Sensitive local records             Minimize collection, encrypt
                                      sensitive local data where
                                      appropriate, provide retention
                                      controls

  Google Play policy rejection        Review VpnService and user-data
                                      disclosure policies before
                                      distribution
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 16. Research and implementation references

Use official documentation as the primary source because Android APIs
and store policies change.

-   [Android VpnService
    guide](https://developer.android.com/develop/connectivity/vpn)
-   [Android VpnService API
    reference](https://developer.android.com/reference/android/net/VpnService)
-   [Android AppOpsManager API
    reference](https://developer.android.com/reference/android/app/AppOpsManager)
-   [Android foreground
    services](https://developer.android.com/develop/background-work/services/foreground-services)
-   [Android foreground-service
    changes](https://developer.android.com/develop/background-work/services/fgs/changes)
-   [Android
    WorkManager](https://developer.android.com/topic/libraries/architecture/workmanager)
-   [Google Play VpnService
    policy](https://support.google.com/googleplay/android-developer/answer/12564964)
-   [Google LiteRT-LM
    overview](https://developers.google.com/edge/litert-lm/overview)
-   [Google LiteRT
    overview](https://developers.google.com/edge/litert/overview)
-   [Android privacy guidance](https://developer.android.com/privacy)

------------------------------------------------------------------------

## 17. Final recommended build strategy

1.  Build a native Kotlin Android app.
2.  Prove permission visibility and network attribution on real devices
    before designing around them.
3.  Build the local database and rules engine.
4.  Add the local VPN monitor with explicit consent and a persistent
    notification.
5.  Build useful app reports without AI.
6.  Integrate a small quantized SLM through a supported Android runtime.
7.  Trigger inference on demand or during scheduled analysis.
8.  Profile performance and test the limits honestly.
9.  Keep the core product offline-first, transparent and useful with the
    SLM disabled.

### Product definition

**A lightweight, offline-first Android privacy intelligence application
that continuously observes available app-permission and network
metadata, correlates those signals with local tracker intelligence, and
uses an optional on-device small language model to explain possible
privacy implications in plain language.**

Its value is not that it can see everything. Its value is that it makes
the evidence Android does expose understandable without sending the
user's activity to a cloud service.
