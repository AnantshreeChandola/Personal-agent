```mermaid
  flowchart LR
  U[User] --> IN[Intake & Reason]
  IN --> RAG((ContextRAG: prefs/history/exemplars))
  IN --> PLIB[Plan Library]
  PLIB --> RET[Retrieve & Score Plans]
  IN --> REG[Plugin Registry]
  REG --> SEL[Select Plugins]
  SEL --> PLAN[Planner dry run plan]
  PLAN --> SIG[Signer Ed25519]
  SIG --> PREV[Preview Orchestrator n8n, read-only]
  PREV -->|Preview card + evidence| U
  U -->|Approve| GATE[Approval Gate]
  GATE --> EXE[n8n Execute short jobs]
  GATE --> DUR[Temporal Durable Orchestrator long jobs]
  EXE --> ADP[Adapters / Activities]
  DUR --> ADP
  ADP --> AUD[Audit & Metrics]
  ADP --> PW[PlanWriter → Plan Library + History]
  AUD --> U
  PW --> RAG
  ```

  context aware, personalized, self learning.