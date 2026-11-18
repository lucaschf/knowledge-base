# Auditoria obrigatória: falhe antes da chamada

Alguns fluxos exigem que **toda** chamada a um serviço externo deixe trilha de auditoria. O caso típico é a autorização de pagamento: numa disputa de cobrança, "não temos registro dessa tentativa" não é uma resposta aceitável. A política costuma se chamar *fail-closed*: se não der para auditar, a operação falha.

O erro comum é aplicar essa política no lugar errado.

!!! note "Outro sentido de fail-closed"
    Na [Anti-Corruption Layer](acl.md#fail-closed-na-duvida-nao-passa), fail-closed é o default de dado na borda: na dúvida, não prometer. Aqui o termo é sobre a trilha de auditoria: sem trilha, não chamar.

## Abortar depois da chamada não garante nada

A implementação ingênua chama o gateway, tenta gravar o registro e, se a gravação falhar, levanta a exceção mesmo que a chamada tenha dado certo:

```python
result = gateway.authorize(order)
try:
    audit_log.append(record_for(result))
except AuditWriteError:
    if policy is FAIL_CLOSED:
        raise  # a autorização já aconteceu
```

Repare no que isso realmente garante:

1. **A chamada já aconteceu.** O pagamento foi autorizado, o dado saiu e o provider cobrou pela requisição. Abortar agora não desfaz nada.
2. **O cliente vê um erro e tenta de novo.** É uma segunda autorização e uma segunda cobrança do provider.
3. **O circuit breaker conta a falha errada.** O breaker envolve o gateway, então a falha de gravação é contada como falha do provider. Com o banco de auditoria fora do ar, os circuitos de todos os gateways auditados abrem, mesmo com os providers saudáveis.
4. **O use case recebe uma exceção que não esperava.** Se ele sabia degradar com erro do gateway, agora quebra com erro de infraestrutura.

O que se queria garantir era "nenhuma chamada sem trilha". Isso só pode ser garantido **antes** da chamada.

## Write-ahead: grave um marcador antes

A solução vem do mesmo princípio de um write-ahead log: registre a intenção antes do efeito.

1. **Antes da chamada**, grave um marcador `pending` com um `call_id` novo e só os metadados: provider, quando, de quem.
2. **Se essa gravação falhar**, aborte: o provider não é chamado, nada foi enviado e nada foi cobrado.
3. **Depois da chamada**, grave o registro final com o mesmo `call_id`, com requisição, resposta, latência e status.
4. **Se a gravação final falhar**, só registre um aviso no log. Daqui em diante, nada levanta exceção.

O marcador e o registro final são **dois registros imutáveis, append-only**, ligados pelo `call_id`. Nenhum documento é atualizado. O registro final tem o mesmo formato de sempre, então relatórios e consultas existentes só precisam ignorar `status = 'pending'`.

Por que não fazer só um insert na abertura e um update no fechamento? Porque isso quebra o contrato append-only da trilha de auditoria e exige upsert para cobrir o caso em que o marcador não foi gravado. Dois registros resolvem o mesmo problema sem alterar nada que já foi escrito.

Por que não verificar se o banco está de pé antes de chamar? Porque ele pode cair entre a verificação e a gravação final. A única prova de que a auditoria funciona é ter gravado algo de fato.

## Só pague a escrita extra onde ela compra algo

O marcador custa uma escrita no caminho crítico. Na política *best-effort* (perder um registro eventual é aceitável), ele não compra garantia nenhuma, porque uma falha nele também só geraria um aviso. Então ele só existe sob fail-closed:

| | Best-effort | Fail-closed |
|---|---|---|
| Escritas por chamada | 1, depois | 2, antes e depois |
| Se a escrita de antes falha | Não se aplica | O gateway **não é chamado** |
| Se a escrita de depois falha | Aviso no log | Aviso no log |
| Chamada interrompida no meio | Não deixa rastro | Deixa um marcador órfão |
| Latência extra antes da chamada | Nenhuma | Um insert |

O marcador órfão (`pending` sem registro final) é a evidência de uma chamada interrompida. Uma consulta de alerta basta para encontrá-los:

```sql
SELECT p.call_id, p.gateway, p.recorded_at
FROM gateway_calls AS p
WHERE p.status = 'pending'
  AND p.recorded_at < now() - interval '5 minutes'
  AND NOT EXISTS (
    SELECT 1 FROM gateway_calls AS done
    WHERE done.call_id = p.call_id AND done.status <> 'pending'
  );
```

O corte de tempo precisa ser maior que o pior caso de timeout do gateway somado às tentativas de retry. Senão a consulta acusa chamadas que ainda estão em andamento.

## Dê um nome próprio ao erro

Quando o marcador não pode ser gravado, levante um erro próprio (algo como `AuditUnavailable`), e não o erro genérico do repositório. Ele precisa ser diferente por dois motivos:

- **HTTP.** Erro de repositório costuma virar 500. Aqui o gateway não foi chamado e nada mudou: é indisponibilidade temporária, **503**, e o cliente pode tentar de novo com segurança.
- **Circuit breaker.** A queda da auditoria não diz nada sobre a saúde do provider, então esse erro deve entrar na lista de exceções que o breaker ignora. Com um tipo próprio, você exclui só ele; excluir o erro genérico do repositório silenciaria qualquer outro erro de banco que passasse pelo breaker.

!!! warning "Exceção excluída pode contar como sucesso"
    Em bibliotecas como o pybreaker (conferido na 1.2.0), uma exceção da lista de exclusão é tratada como **sucesso**. No estado *half-open*, uma falha do marcador fecha o circuito sem que o provider tenha sido testado. O estrago é pequeno, porque a próxima chamada real testa, mas vale saber que acontece.

## Relacionados

- [Auditando chamadas externas com um span](auditoria-de-chamadas-externas.md) — a mecânica que este padrão estende
- [Anti-Corruption Layer](acl.md) — o outro sentido de fail-closed, para dados na borda
- [Os 8 eixos de auditoria](acl-eixos.md) — o que conferir no adapter que fica dentro do decorator
