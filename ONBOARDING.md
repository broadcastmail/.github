## Onboarding

## Graph 1a - C4 Level 3 State validation and user lookup
Question validated: does the callback validate state before touching any user data, and does an existing user always bypass onboarding?
<img width="1110" height="1342" alt="mermaid-diagram-2026-10-30-162907" src="https://github.com/user-attachments/assets/0c13889b-6b02-4ec3-917d-cf152700f6e4" />

## Graph 1b - C4 Level 3 Project count determines onboarding path
Question validated: is the partial session always invalidated before the full session is created, so two sessions can never coexist?
<img width="1154" height="1762" alt="mermaid-diagram-2026-10-30-162946" src="https://github.com/user-attachments/assets/28690a4a-d030-4850-b10f-d7a36d5f963d" />


## Graph 2 — Backend enforces step preconditions, not the frontend
Question validated: if a user hits a step endpoint out of order, is the backend the last line of defense — not the frontend route guard?
<img width="2123" height="1602" alt="mermaid-diagram-2026-10-30-140130" src="https://github.com/user-attachments/assets/26c13ec3-defc-4015-9b39-454ca7dabb08" />

## Graph 3 — C4 Level 2 What POST /onboarding/complete does
Question validated: are all three preconditions checked before any external writes, so a half-completed account is impossible?
<img width="2866" height="1968" alt="mermaid-diagram-2026-10-30-135555" src="https://github.com/user-attachments/assets/fb862d42-d9d7-4a4d-a28e-e6df2a447266" />

## Graph 4 - C4 Level 2 User navigates away and comes back
Question validated: when a user returns mid-onboarding, does the backend determine where they resume — not the frontend?
<img width="2123" height="1602" alt="mermaid-diagram-2026-10-30-140130" src="https://github.com/user-attachments/assets/59bfcefb-463c-4029-a2e7-6f5e00530ee0" />
