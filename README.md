# 🏦 Dueto (Gestão Financeira a Dois)

O **Dueto** é um aplicativo web focado na gestão financeira compartilhada para casais. Projetado com foco na experiência mobile, ele funciona como uma SPA (*Single Page Application*) com visual nativo de aplicativo (estilo iOS).

O grande diferencial do projeto é sua arquitetura híbrida: ele roda de forma 100% instantânea salvando os dados no dispositivo através do `localStorage` e utiliza uma planilha do **Google Sheets** como banco de dados em nuvem gratuito e sem servidor (Serverless), permitindo que duas pessoas mantenham as finanças sincronizadas em tempo real em aparelhos diferentes.

---

## ✨ Funcionalidades Implementadas

### 👫 Perfil e Configuração Local
* **Identificação Isolada:** Cada usuário escolhe no seu próprio aparelho se é a Pessoa A ou a Pessoa B e personaliza os nomes.
* **Armazenamento Seguro:** O link secreto da planilha do Google Sheets é colado diretamente na aba de configurações do aplicativo no seu celular. O código-fonte fica público no GitHub, mas os seus dados ficam privados.

### 💰 Cofres Virtuais com Metas Individuais
* **Criação de Cofres:** Organize as economias criando cofres com nomes e símbolos (emojis) personalizados.
* **Barras de Progresso:** Defina metas financeiras em cada cofre. O app exibe o progresso em tempo real.
* **Histórico de Atividade:** Controle total de depósitos, retiradas e pagamentos recentes.

### 🤝 Empréstimos Internos com Carência
* **Retirada Planejada:** Retire dinheiro de um cofre específico simulando um empréstimo parcelado (ex: 2x, 3x, 6x).
* **Período de Carência:** Agende o início do pagamento para meses futuros (daqui a 1 ou 2 meses), exibindo um alerta vermelho nos detalhes do cofre.

### 📊 Gestão Mensal Inteligente e Rateio
* **Fluxo de Entrada:** Insira seus saldos separados por categorias: Dinheiro Físico, Vale-Alimentação e Conta Bancária.
* **Rateio de Despesas:** Ao pagar uma conta, o app sugere o rateio inteligente (ex: prioriza zerar o Vale-Alimentação antes de usar o dinheiro físico).
* **Despesas Direcionadas a Cofres:** Cadastre contas recorrentes cujo dinheiro pago seja transferido automaticamente para um cofre escolhido.

### 🤖 Motor de Auto-Distribuição de Sobras
* **Injeção Inteligente:** Um botão especial ("✨ Distribuir pelas metas") calcula o que sobrou livre no seu mês e injeta automaticamente esse dinheiro nos cofres que ainda não atingiram as metas.

---

## 🚀 Como Configurar o Banco de Dados (Google Drive)

Para sincronizar os seus dados entre aparelhos e garantir a segurança com backups, precisamos criar o script no Google Drive e **gerar o link de sincronização**. Siga o passo a passo com atenção:

### Passo 1: Criar o Script e Inserir o Código

1. Acesse o [App Script](https://script.google.com/) e crie um **Novo projeto** em branco (ex: `BD_CorridaPlus`).
2. Apague todo o código que estiver na tela e cole este bloco abaixo:

```javascript
const FILE_NAME = "dueto_dados_sync.json"; // Nome do ficheiro da app DUETO
const FOLDER_NAME = "cofre"; // Pasta principal
const BACKUP_FOLDER_NAME = "cofre/cofre_backups"; // Caminho para a subpasta de backups

// ==========================================
// 1. Função Auxiliar (Lida com subpastas corretamente)
// ==========================================
function getOrCreateFolder(folderPath) {
  const parts = folderPath.split('/');
  let currentFolder = DriveApp.getRootFolder();
  
  for (let i = 0; i < parts.length; i++) {
    let folderName = parts[i];
    let folders = currentFolder.getFoldersByName(folderName);
    
    if (folders.hasNext()) {
      currentFolder = folders.next(); // Entra na pasta se ela existir
    } else {
      currentFolder = currentFolder.createFolder(folderName); // Cria a pasta se não existir
    }
  }
  return currentFolder;
}

// ==========================================
// 2. Função POST (Salvar dados)
// ==========================================
function doPost(e) {
  try {
    const data = e.postData.contents;
    const folder = getOrCreateFolder(FOLDER_NAME);
    let files = folder.getFilesByName(FILE_NAME);
    let file;
    
    if (files.hasNext()) {
      file = files.next();
      file.setContent(data);
    } else {
      file = folder.createFile(FILE_NAME, data, MimeType.PLAIN_TEXT);
    }
    
    return ContentService.createTextOutput(JSON.stringify({status: "success"}))
      .setMimeType(ContentService.MimeType.JSON);
      
  } catch(err) {
    return ContentService.createTextOutput(JSON.stringify({error: err.message}))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

// ==========================================
// 3. Função GET (Ler dados)
// ==========================================
function doGet(e) {
  try {
    const folder = getOrCreateFolder(FOLDER_NAME);
    let files = folder.getFilesByName(FILE_NAME);
    
    if (files.hasNext()) {
      let file = files.next();
      let content = file.getBlob().getDataAsString();
      return ContentService.createTextOutput(content)
        .setMimeType(ContentService.MimeType.JSON);
    } else {
      return ContentService.createTextOutput(JSON.stringify({}))
        .setMimeType(ContentService.MimeType.JSON);
    }
    
  } catch(err) {
    return ContentService.createTextOutput(JSON.stringify({error: err.message}))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

// ==========================================
// 4. Função de Backup Diário
// ==========================================
function fazerBackupDiario() {
  const mainFolder = getOrCreateFolder(FOLDER_NAME);
  let files = mainFolder.getFilesByName(FILE_NAME);

  if (files.hasNext()) {
    let originalFile = files.next();
    
    // Pega a data de hoje no formato YYYY-MM-DD
    let dataHoje = Utilities.formatDate(new Date(), Session.getScriptTimeZone(), "yyyy-MM-dd");
    let nomeBackup = "dueto_backup_" + dataHoje + ".json"; 

    // Vai buscar (ou criar) a pasta de destino usando o caminho completo
    const backupFolder = getOrCreateFolder(BACKUP_FOLDER_NAME);
    
    // Verifica se o backup de hoje já existe para evitar cópias duplicadas
    let existingBackups = backupFolder.getFilesByName(nomeBackup);
    if (!existingBackups.hasNext()) {
      // Faz a cópia apenas se não existir um ficheiro com o nome de hoje
      originalFile.makeCopy(nomeBackup, backupFolder);
    }
  }
}
```

3. Clique no ícone de **Salvar** (disquete) no menu superior.

### Passo 2: Configurar o Backup Automático

Para que o sistema faça cópias de segurança sozinho:
1. No menu lateral esquerdo do Apps Script, clique no ícone de relógio (**Acionadores** ou **Triggers**).
2. Clique no botão azul **Adicionar acionador**.
3. Configure da seguinte forma:
   * **Escolha a função que será executada:** `fazerBackupDiario`
   * **Selecione a origem do evento:** `Baseado no tempo`
   * **Selecione o tipo de acionador com base no tempo:** `Temporizador diário`
   * **Selecione a hora:** Escolha um horário de sua preferência (ex: *Meia-noite a 1h da manhã*).
4. Clique em **Salvar** (se pedir autorização, permita o acesso à sua conta).

### Passo 3: Como Gerar e Pegar o Link (Atenção Aqui!)

Este é o momento de criar a URL que o seu aplicativo vai usar.
1. No canto superior direito do Apps Script, clique no botão azul **Implantar** e escolha **Nova implantação**.
2. Clique no ícone de engrenagem (`⚙️`) ao lado de "Selecione o tipo" e escolha **App da Web**.
3. Preencha os campos **EXATAMENTE** desta forma:
   * **Descrição:** Pode escrever "Versão 1".
   * **Executar como:** *Você (seu-email@gmail.com)*
   * **Quem tem acesso:** Selecione **Qualquer pessoa** *(Se não marcar "Qualquer pessoa", o celular não vai conseguir salvar)*.
4. Clique no botão azul **Implantar**.
5. Vai aparecer um aviso de "Autorização necessária". Clique em **Autorizar acessos** e escolha o seu e-mail.
6. O Google mostrará uma tela dizendo "O Google não verificou este app". Clique na palavra **Avançado** (lá embaixo) e depois clique em **Acessar projeto sem título (não seguro)**. Clique em **Permitir**.
7. Na última tela que aparecer, você verá escrito "URL do app da Web" e um link gigante embaixo (terminando com `/exec`). **Copie este link gigante.**

### Passo 4: Colar o Link no Aplicativo

1. Abra o site do seu aplicativo no seu celular.
2. Navegue até a aba inferior direita chamada **Você** (Configurações).
3. Procure o campo **"URL de Sincronização"** e cole o link gigante lá dentro.
4. Defina a sua taxa de **IVA** e de **Comissão** nas configurações (caso aplicável).
5. Pronto! Faça o mesmo em outro aparelho com o mesmo link para manter os dados sincronizados.

---

## 🛠️ Performance e Sincronização Manual
Para não estourar o limite de acessos diários do Google:
* O app salva localmente e só envia dados para a nuvem alguns segundos após você terminar de fazer uma alteração.
* Ele puxa dados novos automaticamente apenas quando o aplicativo é aberto.
* **Quer puxar atualizações manualmente?** Basta tocar na pílula escrita "sincronizado" ou "offline" no topo do app para forçar a atualização da tela naquele momento.

---

## 📱 Dica: Instale no Celular como um App Real
Abra o site no navegador do celular (Safari ou Chrome), clique no botão de **Compartilhar** do navegador e escolha **"Adicionar à Tela de Início"**. O navegador ocultará as barras de endereço e o ícone ficará junto com seus outros aplicativos.

