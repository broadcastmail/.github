# Architecture

## Overveiw
C2 system diagram — services, external dependencies, data stores
<img width="3088" height="1051" alt="broadcastmai-architecture" src="https://github.com/user-attachments/assets/38b2f74a-bb6c-4bb5-8cc2-7a3870e45d16" />

## Campaign lifecycle
Campaign transitions are derived from the worker pooler (not pushed or event-driven). Simple and reliable for V1.
<img width="2033" height="1402" alt="camapign-states" src="https://github.com/user-attachments/assets/4bd4c73e-4e2b-48da-b1d1-2922e6e49189" />

## Reliable delivery (outbox pattern)
The outbox pattern claims rows before sending. Idempotent sends when multiple instances are being run. Needed because claims and sends are separate transactions.
<img width="5701" height="802" alt="broadcastmail-outbox" src="https://github.com/user-attachments/assets/a5f94f43-7b14-4d24-8719-6b3e0993bdc4" />

## Plan enforcement
C3 Graph - Behaviour per plan is defined once, enforced via Facade. Adding a new plan tier means one new strategy class and one enum constant. No existing code changes.

<img width="500" height="850" alt="broadcast-plan-enforcement" src="https://github.com/user-attachments/assets/ca839b05-d9fb-4c26-a2dc-67bbb74ede1b" />
