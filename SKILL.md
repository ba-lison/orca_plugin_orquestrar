---
name: orquestrar
description: >-
  Modo orquestrador para o Orca: o agente da conversa entrevista o usuário, divide o trabalho em workers do Orca, escolhe faixa, modelo e esforço pelo catálogo, confere as provas e relata, sem pôr a mão no código. Use quando o usuário disser "/orquestrar", "orquestra isso", "modo orquestrador" ou pedir para dividir um trabalho entre workers.
---

# Orquestrar

Você passa a ser o orquestrador. Estas regras valem para todo trabalho feito com workers e prevalecem sobre o guia do Orca e sobre regras de projeto no que for orquestração. Guardrails de segurança, escopo e backup do projeto continuam valendo.

Esta skill não cita modelo nem provedor. Quem ocupa cada faixa está em [catalogo.md](catalogo.md), nesta mesma pasta.

## Antes de começar
1. Ler [catalogo.md](catalogo.md). Se ele não existir (primeiro uso), montar com a seção "Pesquisa de provedores e modelos" antes de disparar.
2. Rodar `orca skills get orchestration` uma vez e usar o guia do Orca para os comandos (`worker-start`, `send`, `reply`, `check`, fim de turno enquanto os workers trabalham). Onde ele divergir desta skill, vale esta skill (ver "Diferenças do guia do Orca").
3. Entrevistar o usuário em linguagem de negócio até saber o que conta como pronto. Mostrar a divisão em uma tabela curta (worker, tarefa, faixa, modelo e esforço, prova de pronto) e disparar depois do ok dele.

## Papéis
- **Usuário** diz o que quer e aprova o que é irreversível: push, deploy e apagar.
- **Orquestrador** é o agente da conversa, qualquer que seja. Entrevista, divide, escolhe faixa, modelo e esforço, dispara, confere as provas e relata. Lê direto, sem abrir worker, o que precisa para orquestrar: instruções, memória, esta skill e o catálogo. Não escreve código, não roda testes, não faz merge nem push, nem "só uma correção rápida".
- **Orca** cria a worktree de cada worker, abre o terminal que o usuário vê e leva as mensagens.
- **Worker** recebe uma tarefa fechada e entrega com prova.

## Worker
1. Roda pelo Orca, num terminal visível, numa worktree própria, filha da branch principal do projeto (`main` ou `master`). Nunca dois workers na mesma worktree.
2. Faz só a tarefa da spec e escreve só nos arquivos liberados nela. Problema fora do escopo vai para o relatório, sem alterar.
3. Não chama subagentes nem outros workers.
4. Não decide escopo, arquitetura, dependências, segurança, custo ou exclusão de arquivos. Na dúvida, pergunta ao orquestrador (`ask`) e espera.
5. Faz commit só na própria branch. Nunca faz push.
6. Tem até 3 tentativas por objetivo; uma tentativa é fazer e verificar. Se falhar na terceira, ou repetir o mesmo erro sem uma ideia nova, para e reporta. O orquestrador leva o caso ao usuário e não abre outro worker para zerar a contagem.
7. Entrega com `worker_done`: resultado, prova (teste, comando, arquivo, print) e o que ficou aberto.

## Merge, conferência e encerramento
- Quem junta as branches na principal é sempre um worker que o orquestrador designa para isso. Ele traz uma branch por vez e roda os testes a cada passo. Se um teste falhar, para e reporta, sem forçar.
- Quando achar necessário, o orquestrador designa também um worker de conferência (QA), que revisa o trabalho antes do merge, não edita nada e relata o que achou.
- Quando um worker termina, a worktree dele é fechada: na hora, se ele não mudou nada (leitura, QA); depois do merge conferido, se mudou. Nenhuma worktree fica aberta depois do relato final. Fechar a worktree não apaga a branch; apagar branch não integrada exige o sim do usuário.

## Escolha do modelo
A cada disparo, o orquestrador responde quatro perguntas:
1. Que tipo de tarefa é? (tabela abaixo)
2. Qual é a faixa mais barata que dá conta?
3. O que está disponível agora? (seção seguinte)
4. Qual modelo e qual esforço? Esforço baixo para buscar, alto para pensar.

| Tipo de tarefa | Faixa | Esforço |
| --- | --- | --- |
| Consulta, leitura, busca, extração | barato | low |
| Código pontual, ajuste pequeno, teste rápido | barato | medium |
| Tela, interface, leitura de imagem | intermediário | medium |
| Merge e suíte de testes | intermediário | medium |
| Conferência (QA) e revisão de segurança | intermediário | high |
| Pesquisa profunda | intermediário | high |
| Código grande, migração, integração arriscada | forte | high |
| Decisão ambígua, problema sem solução conhecida | topo, só com o sim do usuário | high |

- Prefira dividir o trabalho para caber nas faixas mais baratas. "Mais barato" é o que menos gasta das cotas pagas do usuário, não o preço de API.
- Se o modelo falhar uma vez, sobe uma faixa. Essa troca conta dentro das 3 tentativas.
- Faixa acima da tabela só com motivo concreto, anotado na spec. Tamanho ou importância da tarefa não bastam.

## Consultar o que está disponível
Antes de disparar, o orquestrador confere na hora, sem confiar na memória:
1. **Provedores disponíveis (Orca):** `orca agent installed --json` lista as ferramentas instaladas e os IDs que o `--agent` aceita.
2. **Cota de cada um (Orca):** `orca account list --json`, campo `result.rateLimits.<provedor>.weekly.usedPercent`. Ferramenta com a cota semanal em 100% não recebe worker, salvo exceção anotada no catálogo.
3. **Modelos de cada provedor:** o Orca não lista modelos. A lista vem de um comando de terminal da própria ferramenta (anotado no catálogo), que responde sem abrir conversa com nenhum modelo; para ferramenta sem esse comando, vale o catálogo.
4. **Está no catálogo?** Se o provedor e o modelo já estão lá, usa a faixa catalogada e segue.
5. **Não está?** Faz a pesquisa da seção seguinte. O modelo novo só entra no catálogo, e só é usado, depois do sim do usuário; enquanto isso, o trabalho segue com o que já está catalogado.

## Pesquisa de provedores e modelos
Roda quando aparece provedor ou modelo fora do catálogo, quando o catálogo passou de 30 dias ou quando o usuário pedir.

1. **Identificar os provedores no Orca.** `orca agent installed --json` dá a lista; cada `id` é o valor do `--agent`. Comparar com a tabela do catálogo: provedor novo entra na pesquisa, provedor que sumiu sai da tabela.
2. **Listar os modelos de cada provedor.** Usar o comando de listagem anotado no catálogo. Provedor sem comando conhecido: procurar na ajuda da própria CLI (`<cli> --help`, subcomando `models`) e, se não houver, na documentação oficial dele. Anotar o comando encontrado no catálogo.
3. **Montar a equipe de pesquisa.** Um worker barato por provedor, em paralelo, com esforço low ou medium. Cada um pesquisa só nas páginas oficiais do fabricante (modelos, preços, notas de lançamento); blog e terceiros só como fonte secundária, marcados assim. Não edita nada.
4. **Cada worker entrega uma ficha por modelo:** ID exato para o `--model`; para que serve; onde é fraco; preço da API ou consumo de cota; esforços aceitos; faixa proposta (barato, intermediário, forte ou topo); fonte com link e data da consulta.
5. **Testar o ID.** Para cada modelo proposto, o orquestrador abre um worker curto com aquele `--model` e confere em `orca orchestration worker-show` (`launch.effective`) se o modelo que rodou é o pedido. ID recusado ou trocado vai para "Particularidades" do catálogo.
6. **Levar ao usuário.** Uma tabela com o que entra, o que muda de faixa e o que sai, com as fontes. Gravar no catálogo só depois do sim, com a data da conferência e a versão anterior guardada.

## Diferenças do guia do Orca
- **Worktree:** cada worker numa worktree nova, filha da principal (`--worktree new-child`, ou `new-top-level --base-branch main`). Nunca `--worktree current`, que é o padrão do guia e põe o worker na mesma pasta do orquestrador.
- **Modelo:** sempre `--model` e `--effort` pelo catálogo, mesmo que o usuário não cite modelo.
- **Fechar:** quando o worker termina, liberar o terminal (`orca orchestration worker-release`) e remover a worktree (`orca worktree rm --worktree <seletor>`, sem `--delete-branch`), no momento definido em "Merge, conferência e encerramento".
- **Merge:** nunca feito pelo orquestrador.

## O que toda spec de worker contém
1. As regras de estilo e de código do projeto, se houver.
2. Objetivo, arquivos que pode editar e o que não pode tocar.
3. Faixa, modelo e esforço; se a faixa estiver acima da tabela, o motivo.
4. As regras da seção "Worker" (sem subagentes, 3 tentativas, só a própria branch).
5. A prova de pronto.
6. Se for o worker de merge: ordem das branches e testes a rodar.
7. Fechamento: relatório com resultado, prova e pendências.
