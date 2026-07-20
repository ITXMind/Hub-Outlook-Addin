# Hub – Anexar Email (Outlook Add-in)

Add-in do Outlook do **ITXMind Hub** que permite, diretamente a partir de um email:

- **Anexar o email** a uma oportunidade (deal) já existente no Hub.
- **Criar uma nova oportunidade** a partir do email (pré-preenchendo o título com o assunto), e anexar o email de seguida.

O add-in corre inteiramente no lado do cliente (task pane HTML/JS), comunicando diretamente com o Supabase do Hub — não tem backend próprio, à exceção de uma Edge Function usada para operações que exigem permissões elevadas.

---

## Estrutura do repositório

```
.
├── manifest.xml              # Manifesto do add-in (registo no Outlook)
└── taskpane/
    ├── index.html             # UI principal (anexar / criar oportunidade)
    ├── commands.html          # Ponto de entrada para comandos do ribbon
    ├── auth-dialog.html        # Diálogo de autenticação (login Supabase)
    └── assets/                # Ícones do add-in (16/32/80px)
```

Este repositório (`HostingTaskpane`) é o que é efetivamente **publicado via GitHub Pages** e carregado pelo Outlook em runtime. É o único sítio que precisa de estar atualizado para o add-in refletir alterações — outras cópias/repos de desenvolvimento não têm efeito até serem copiadas para aqui.

- URL público: `https://afonsocanato.github.io/HostingTaskpane/`
- Task pane carregado pelo Outlook: `https://afonsocanato.github.io/HostingTaskpane/taskpane/index.html?v=2`

---

## Funcionalidades

### 1. Anexar email a oportunidade existente
- Pesquisa de oportunidades (`deals`) por título/cliente.
- Ao selecionar, o email (corpo + anexos) é enviado para a Edge Function `attach-email-to-deal`, que:
  - Faz upload do email e dos anexos para o bucket `deal-attachments`.
  - Regista as entradas em `deal_attachments` e `deal_email_attachments`.

### 2. Criar oportunidade a partir do email
- Pré-preenche o **título** com o assunto do email.
- Pesquisa e seleção de **cliente** e **contacto** (contacto só aparece depois de escolher o cliente; se o cliente só tiver 1 contacto, é auto-selecionado).
- Campo de **valor estimado**.
- Campo **"Data prevista de fecho"** (`expected_close_date`) — **obrigatório**, com um calendário próprio (não usa o `<input type="date">` nativo do browser, para manter consistência visual entre Windows/Mac e alinhado ao resto do design do Hub).
- Ao submeter:
  1. Reserva o próximo número de oportunidade (`reserve_next_opp_number`).
  2. Cria o registo em `deals` (stage inicial `qualification`, probabilidade `10%`).
  3. Se a criação falhar, liberta o número reservado (`release_opp_number`).
  4. Sucesso → oferece logo anexar o mesmo email à oportunidade recém-criada.

---

## Stack técnica

- **HTML/CSS/JS puro** (sem framework) — mantém o add-in leve e rápido a carregar.
- **Office.js** — integração com o Outlook (leitura do email selecionado).
- **Supabase JS SDK** (`@supabase/supabase-js`) — autenticação e acesso direto à base de dados (`deals`, `clients`, `contacts`, `deal_attachments`, etc.), com RLS a aplicar as permissões.
- **Edge Function** `attach-email-to-deal` — usada para operações que precisam de mais do que o cliente anon consegue fazer (ex.: escrita cruzada de tabelas relacionadas).

---

## Desenvolvimento local

Este projeto não tem build step — é HTML/JS servido estaticamente. Para testar alterações:

1. Edita `taskpane/index.html` (ou os outros `.html`) diretamente.
2. Faz commit/push para a branch usada pelo GitHub Pages (normalmente `main`).
3. Aguarda a republicação do Pages (~1-2 min).
4. No Outlook, força o refresh do add-in:
   - Fecha completamente o Outlook e reabre.
   - Se persistir cache antiga, limpa a cache do WebView:
     - **Windows:** `%LOCALAPPDATA%\Microsoft\Office\16.0\Wef\`
     - **Mac:** `~/Library/Containers/com.microsoft.Outlook/Data/Documents/wef/` e/ou `~/Library/Caches/com.microsoft.Outlook`
   - Em último caso, incrementa o parâmetro `?v=` no `manifest.xml` (`SourceLocation` / `bt:Url`) para forçar o Outlook a tratar o recurso como novo, e volta a fazer sideload do manifesto.

### Como confirmar que o deploy propagou
Abre diretamente o URL do `index.html` num browser normal (não no Outlook) e faz hard refresh (`Cmd/Ctrl+Shift+R`). Se a alteração não aparecer aí, o problema é do deploy/GitHub Pages, não de cache do Outlook.

---

## Instalação do add-in (sideload)

1. Faz download do `manifest.xml` deste repositório.
2. No Outlook: **Ficheiro → Obter Suplementos → Meus suplementos → Adicionar do ficheiro** (ou equivalente na versão web/Mac).
3. Seleciona o `manifest.xml`.
4. Abre um email — deve aparecer o botão **"Hub - Anexar Email"** no ribbon.

---

## Notas de segurança

- A `SUPABASE_ANON_KEY` usada no `index.html` é a chave pública (anon) — o acesso real aos dados é controlado por Row Level Security (RLS) no Supabase, não pela ocultação desta chave.
- Qualquer alteração a permissões de escrita (ex.: criação de deals, upload de anexos) deve ser validada também do lado das políticas RLS/Edge Function, não apenas na UI.
