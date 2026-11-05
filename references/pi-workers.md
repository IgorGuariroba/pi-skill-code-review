# Workers no Pi

Referência compartilhada para delegação em processos Pi independentes. Um **worker** é outro processo Pi com contexto próprio; o coordenador continua responsável pelo resultado.

## Preparar e executar

1. Confira `pi --help` para as opções da instalação. Crie um diretório de execução fora do repositório (`mktemp -d`) e um prompt por tarefa. Registre papel, cwd absoluto, escopo permitido, referências absolutas à spec/tickets/skills, resultado esperado e critério de conclusão. Compartilhe ponteiros, não o histórico da conversa.
2. Execute cada tarefa com cwd explícito e contexto novo. Exemplo de comando para `process_start` (substitua os caminhos):

   ```bash
   cd /absolute/worktree && pi --print --no-session @/absolute/run/task.md > /absolute/run/result.md 2> /absolute/run/stderr.log
   ```

   Use o modelo/provider solicitado, se houver; caso contrário, mantenha os defaults configurados. `--no-session` evita persistir/reutilizar a conversa; configurações e instruções do projeto continuam disponíveis. Carregue outras skills lendo seus `SKILL.md` por caminho absoluto no prompt. Pi não precisa de uma ferramenta `Skill` para isso.
3. Inicie tarefas independentes com chamadas separadas de `process_start`, cada uma com `watch`: `success: {exitCode: {eq: 0}}`, `failure: {exitCode: {ne: 0}}` e `timeoutMs` finito conforme o trabalho (por exemplo, 600000 para revisão). Após registrar os watches, informe a espera e encerre o turno. Os resultados chegam automaticamente; use `process_logs` para diagnóstico, não polling manual.
4. Ao receber a conclusão, leia resultado e stderr e confira os artefatos. Exit code zero comprova execução, não cumprimento da tarefa. Uma resposta sem evidência, com bloqueio ou trabalho parcial é incompleta. Em timeout, inspecione e pare o processo antes de repetir ou liberar seu worktree. Preserve prompts, resultados e logs até entregar o trabalho.

Se `process_start`/watch não estiverem disponíveis, escolha o caminho Herdr abaixo quando aplicável; caso contrário, execute tarefas sequencialmente com timeout explícito na ferramenta disponível. Registre a perda de paralelismo. Falha de autenticação ou diálogo que exija interação deve ser reportado ao usuário; não invente aprovações nem transfira segredos em prompts.

## Contratos e isolamento

- **Implementação:** um worktree e branch exclusivos por worker escritor. O resultado informa commit(s), critérios atendidos, comandos/testes e resultados, pendências e status Git. Só aceite conclusão após conferir esses itens.
- **Revisão/exploração:** leitura apenas do código; saída final em stdout, capturada pelo coordenador. Autorize escrita de notas externas apenas quando a tarefa exigir. Com `bash`, uma instrução de leitura não é sandbox: mantenha o escopo explícito e verifique o status Git antes/depois. Para leitura estrita sem shell, a instalação oferece `--tools read,grep,find,ls --no-mcp`; o coordenador pode fornecer diff e commits em arquivos externos.
- **Integração:** um único escritor na branch de integração. O coordenador serializa merges; workers de tickets não atualizam essa branch. Nenhum reset destrutivo ou limpeza forçada de alterações alheias.
- **Conclusão:** o coordenador valida commits, testes e critérios de aceitação. Estado de processo/pane não substitui essa validação.

## Herdr: acompanhamento visível

Quando o usuário mencionar/pedir Herdr e `HERDR_ENV=1`, leia `../skills/herdr/SKILL.md` (relativo a este arquivo) e consulte a CLI instalada antes de controlar panes. Fora do Herdr, continue com processos Pi; Herdr não é pré-requisito.

Use os comandos documentados de pane para executar o mesmo comando `pi --print --no-session` em um shell disponível, preservando foco e cwd explícito. Capture saída, stderr e exit code em arquivos exclusivos fora do repositório. Aguarde via `watch_create` com polling barato do arquivo de exit code, sucesso zero, falha não zero e timeout finito. Publique esse arquivo apenas depois que o Pi terminar. Após registrar o watch, encerre o turno. Em timeout, inspecione o pane antes de repetir.

Se preferir um Pi interativo reconhecido por Herdr, confira primeiro se a CLI instalada oferece esse kind e siga a skill Herdr para iniciar, enviar prompt e aguardar. Cada tarefa independente começa em um processo/sessão nova. `idle`/`done` indica estado da interface, não aceitação do trabalho: leia a resposta e valide os artefatos. Crie workspaces/tabs ou outra topologia somente quando o usuário os solicitar; encerre apenas recursos criados para esta execução.
