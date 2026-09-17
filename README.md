<p align="center">
  <img src="logo.png" alt="OFFELLIA" width="320">
</p>

# OFFELLIA_CLI

Console agêntico de desenvolvimento. Um único binário nativo em C que fala com o `llama-server`, executa ferramentas reais no sistema e consulta uma base Dev embutida (FTS5).

**Binário:** `offellia_cli.bin` (~281 MB, Linux x86-64, base Dev embutida). v2.6.0 grava projetos no disco em etapas (tools locais no CWD).

---

## Instalação

O executável é versionado com [Git LFS](https://git-lfs.com). Sem o LFS o clone baixa só um ponteiro e o binário não roda.

```bash
git lfs install
git clone https://github.com/brunoconta1980-tech/OFFELLIA_CLI.git
cd OFFELLIA_CLI
chmod +x offellia_cli.bin
```

Requisito de runtime: `libcurl` (pacote `libcurl4` nas distros Debian/Ubuntu).

---

## Características

- **Agente autônomo** — loop de *tool calling* até concluir a tarefa (`finish_reason: stop`); padrão 64 turnos (`/turns`)
- **Disco em etapas** — `write_file`, `edit_file`, `read_file`, `exec_shell_command`, glob e grep rodam **no processo da CLI**, no `--cwd`. Não passam pelo isolate do llama-server, não têm teto de 60s/16 KB, e o artefato não vai no chat
- **Sem timeout de 10 min no stream** — o curl não mata gerações longas; keepalive + heartbeat enquanto o prompt é avaliado. Se o modelo colar código no chat, a CLI desvia para `offellia_out/` e pede para continuar em tools
- **llama-server** — auto-detecta `8080` ou `5173`; qualquer porta com `-p`
- **Tools extras do servidor** — MCP / demais tools de `GET /tools` seguem em `POST /tools`
- **Base Dev embutida** — ~76 000 chunks de documentação de programação (SQLite FTS5 / BM25) compilados no executável; não exige arquivo `.db` ao lado
- **Tool local** `search_dev_knowledge` — o modelo consulta a base quando precisa de sintaxe, padrões, frameworks ou boas práticas
- **SSE** — resposta em streaming, inclusive `reasoning_content` (o dump de fonte no terminal é truncado)
- **CORS e CWD** — cabeçalhos `Origin` e `x-tool-cwd` nas tools remotas
- **Dependência de runtime** — `libcurl` (SQLite e a base vão dentro do binário)

---

## 1. Subir o llama-server

O servidor precisa de `--agent` e `--tools all`. Exemplo alinhado a este ambiente (porta **5173**):

```bash
export LD_LIBRARY_PATH="/home/userk21/llama_server_VULLKAN_5150/build/bin:${LD_LIBRARY_PATH}"

"/home/userk21/llama_server_VULLKAN_5150/build/bin/llama-server" \
  -m "/home/userk21/Área de trabalho/userk21/LLMS/ΩFFΣLLIα_MXFP4_MOE_gemma-4-26B-A4B-it-ultra-uncensored-heretic-BF16.gguf" \
  -ngl 12 --n-cpu-moe 12 \
  -c 50000 \
  -ctk q8_0 \
  -ctv q8_0 \
  -t 4 \
  -tb 4 \
  -b 2048 \
  -ub 1024 \
  -fa on \
  --cpu-strict 1 \
  --parallel 1 \
  --agent \
  --tools all \
  --reasoning auto \
  --kv-unified \
  --load-mode mmap \
  --cors-origins "*" \
  --webui-mcp-proxy \
  --threads-http -1 \
  --port 5173 \
  --host 127.0.0.1
```

A CLI também aceita a porta `8080` (ou outra) se o servidor estiver nela.

---

## 2. Iniciar a CLI

```bash
cd /home/userk21/OFFELLIA_CLI

# Auto-detecta 8080 ou 5173
./offellia_cli.bin

# Porta explícita (recomendado com o comando acima)
./offellia_cli.bin -p 5173

# Projeto como diretório de trabalho das tools
./offellia_cli.bin -p 5173 --cwd /home/userk21/meu_projeto

# Host / CORS personalizados
./offellia_cli.bin -h 127.0.0.1 -p 5173 --cors http://localhost:5173
```

### Opções

| Opção | Descrição |
| :--- | :--- |
| `-p`, `--port <porta>` | Porta do llama-server (padrão: auto 8080 / 5173) |
| `-h`, `--host <ip>` | Host (padrão: `127.0.0.1`) |
| `-m`, `--model <id>` | Modelo (padrão: o primeiro de `/v1/models`) |
| `-d`, `--db <caminho>` | SQLite externo; omitido, usa a base embutida |
| `--cwd <dir>` | CWD enviado às tools (`x-tool-cwd`) |
| `--cors <origem>` | Cabeçalho `Origin` (padrão: `http://localhost:<porta>`) |
| `--help` | Ajuda |

---

## 3. Comandos no console

| Comando | Descrição |
| :--- | :--- |
| `/tools` | Lista as ferramentas detectadas |
| `/cwd [dir]` | Mostra ou altera o diretório das tools |
| `/agent [on\|off]` | Liga ou desliga a execução autônoma |
| `/turns <n>` | Limite de turnos agênticos (padrão: 64, máx. 200) |
| `/port <porta>` | Troca a porta e recarrega as tools |
| `/host <ip>` | Troca o host e recarrega as tools |
| `/model` | Modelo ativo em `/v1/models` |
| `/search <termo>` | Busca BM25 na base Dev |
| `/clear` | Limpa a tela e redesenha o banner |
| `/help` | Esta lista |
| `/exit` ou `/quit` | Encerra |

Qualquer outra linha é enviada ao modelo como prompt. O agente invoca tools, mostra o raciocínio e itera até validar o resultado no ambiente.

---

## 4. Fluxo agêntico

1. `GET /v1/models` — identifica o modelo carregado.
2. `GET /tools` — carrega os schemas do llama-server e registra `search_dev_knowledge`.
3. `POST /v1/chat/completions` (SSE, `stream: true`, campo `tools`, `tool_choice: auto`). Sem timeout total — gerações longas e prompt de 50k ctx não derrubam a conexão.
4. Se houver `tool_calls`, a CLI executa **localmente** no CWD (`write_file` / `edit_file` / `exec_shell_command` / …). Tools desconhecidas vão a `POST /tools`. O retorno entra no histórico (`role: tool`) e o passo 3 se repete.
5. Código colado no chat (em vez de tool) é gravado em `offellia_out/` e o loop pede continuidade em etapas.
6. Termina quando o modelo responde sem tools (`finish_reason: stop`) e sem dump de fonte.
