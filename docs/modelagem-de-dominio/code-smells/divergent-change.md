# Divergent Change

**Princípio violado:** coesão / acoplamento — um mesmo arquivo/classe muda por muitos motivos diferentes (viola SRP).

## O que é

Uma classe que você edita por razões não relacionadas: hoje por causa de regra fiscal, amanhã por causa de formato de relatório, depois por integração externa. Responsabilidades demais num lugar só. É o oposto do [Shotgun Surgery](shotgun-surgery.md).

## Como detectar

- "Toda mudança no sistema passa por essa classe."
- Histórico do git mostra o mesmo arquivo mudando por motivos totalmente distintos.
- Classe com seções que não conversam entre si.

## Refactoring

- **Extract Class** — separar por eixo de mudança; cada classe muda por um motivo só.
- Alinhar com bounded contexts: motivos de mudança diferentes geralmente são responsabilidades de contextos diferentes.

## Antes / depois

Uma `UserService` que faz auth, billing e notificação → quebrada em `Authentication`, `Billing`, `Notification`. Cada uma muda por seu próprio motivo.

## Quando NÃO marcar

- Classe pequena e coesa que por acaso mudou duas vezes seguidas — divergent change é sobre **eixos** distintos de mudança, não frequência.

## Relacionados

- [Shotgun Surgery](shotgun-surgery.md) — o espelho deste smell.
- [Balanceamento](../../acoplamento/balanceamento.md) — o mesmo problema no nível de módulo, onde aparece como complexidade local.
