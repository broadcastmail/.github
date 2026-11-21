## Onboarding

## Graph 1a - C4 Level 3 State validation and user lookup
Question validated: does the callback check state and redirect origin before it exchanges the code or touches any user data, and does a project that's already connected always skip onboarding?
 ```mermaid
flowchart TD
    A["POST /oauth/supabase/callback"] --> B{"state matches +<br/>redirect origin allowed?"}
    B -- no --> E1["401 / 400 — nothing touched"]
    B -- yes --> X["PKCE token exchange<br/>+ listProjects"]
    X --> M{"any projects?"}
    M -- no --> E2["422 NoSupabaseProjects"]
    M -- yes --> N{"reconfigure intent<br/>with valid bm_session?"}
    N -- yes --> R["→ Graph 1b (reconfigure)"]
    N -- no --> P{"any project already<br/>has a Connection?"}
    P -- yes --> Q["rotateApiKey → RETURNING_USER<br/>→ /dashboard"]
    P -- no --> NU["→ Graph 1b (new account)"]
```

## Graph 1b - C4 Level 3 Project count determines onboarding path
Question validated: is the partial session invalidated only after the full session is created, so a failed project selection can be retried without redoing OAuth?
```mermaid
flowchart TD
    A["from Graph 1a<br/>new account or reconfigure"] --> B{"how many projects?"}
    B -- one --> S["create full session<br/>(reconfigure: reuse role if same project)"]
    S --> O1["200 NEW_USER / RECONFIGURE<br/>+ onboardingToken"]
    B -- many --> P["createPartial<br/>stores project list, 30m TTL"]
    P --> UI["/onboarding/select-project<br/>GET /projects"]
    UI -- "POST /select-project" --> F["create full session<br/>(same setup as 'one')"]
    F --> I["invalidatePartial"]
    I --> O2["set onboarding_session cookie<br/>→ CONFIRM_SCHEMA"]
    F -. "setup fails" .-> K["partial kept → user can retry pick"]
```


## Graph 2 — Backend enforces step preconditions, not the frontend
Question validated: if a user hits a step endpoint out of order, is the backend the last line of defense — not the frontend route guard?
<img width="2123" height="1602" alt="mermaid-diagram-2026-10-30-140130" src="https://github.com/user-attachments/assets/26c13ec3-defc-4015-9b39-454ca7dabb08" />

## Graph 3 — C4 Level 2 What POST /onboarding/complete does
Question validated: are all three preconditions checked before any external writes, so a half-completed account is impossible?
<img width="2866" height="1968" alt="mermaid-diagram-2026-10-30-135555" src="https://github.com/user-attachments/assets/fb862d42-d9d7-4a4d-a28e-e6df2a447266" />

## Graph 4 - C4 Level 2 User navigates away and comes back
Question validated: when a user returns mid-onboarding, does the backend determine where they resume — not the frontend?
<img width="2123" height="1602" alt="mermaid-diagram-2026-10-30-140130" src="https://github.com/user-attachments/assets/59bfcefb-463c-4029-a2e7-6f5e00530ee0" />
