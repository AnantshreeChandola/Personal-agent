```mermaid
  flowchart LR
  U[User] --> IN[Intake & Reason]
  IN --> RAG((ContextRAG: prefs/history/exemplars))
  IN --> PLIB[Plan Library]
  PLIB --> RET[Retrieve & Score Plans]
  IN --> REG[Plugin Registry- n8n bindings]
  REG --> SEL[Select Tools]
  SEL --> PLAN[Planner dry_run plan]
  PLAN --> SIG[Signer Ed25519]
  SIG --> PREV[Preview Orchestrator n8n, read-only]
  PREV -->|Preview card + evidence| U
  U -->|Approve Gate A/B/…| GATE[Approval Gates]
  GATE --> EXE[n8n Execute short jobs via connectors]
  GATE --> DUR[Temporal Durable Orchestrator long jobs]
  EXE --> BIND[Binding Resolver plan → n8n node params]
  EXE --> CONN[n8n Connector Nodes GCal/Gmail/HTTP/Slack…]
  DUR --> ACT[Temporal Activities HTTP/SDK calls as needed]
  CONN --> AUD[Audit & Metrics]
  ACT --> AUD
  CONN --> PW[PlanWriter → Plan Library + History]
  ACT --> PW
  AUD --> U
  PW --> RAG
  ```

  context aware, personalized, self learning.