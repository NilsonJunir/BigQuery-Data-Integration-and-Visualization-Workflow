Aplicativo WEB com IA para appscript
---CODIGO.GS

function doGet() {
  return HtmlService.createTemplateFromFile('Index')
    .evaluate()
    .setTitle('💰 FinanceApp - Nilson Bezerra')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL)
    .addMetaTag('viewport', 'width=device-width, initial-scale=1');
}

function include(filename) {
  return HtmlService.createHtmlOutputFromFile(filename).getContent();
}

function getUsuarioLogado() {
  try {
    let email = Session.getActiveUser().getEmail();

    if (!email || email === '') {
      email = Session.getEffectiveUser().getEmail();
    }

    if (!email || email === '') return null;

    const nome = email.split('@')[0]
      .replace(/[._]/g, ' ')
      .replace(/\b\w/g, c => c.toUpperCase());

    return {
      email,
      nome,
      avatar: nome.charAt(0).toUpperCase()
    };
  } catch (e) {
    Logger.log('Erro getUsuarioLogado: ' + e.message);
    return null;
  }
}

function getUserKey() {
  let email = '';

  try { email = Session.getActiveUser().getEmail(); } catch (e) {}

  if (!email || email === '') {
    try { email = Session.getEffectiveUser().getEmail(); } catch (e) {}
  }

  if (!email) throw new Error('Usuário não autenticado.');
  return email.replace(/[@.]/g, '_');
}

function getAuthUrl() {
  try {
    const authInfo = ScriptApp.getAuthorizationInfo(ScriptApp.AuthMode.FULL);
    if (authInfo.getAuthorizationStatus() === ScriptApp.AuthorizationStatus.REQUIRED) {
      return authInfo.getAuthorizationUrl();
    }
    return null;
  } catch (e) {
    return null;
  }
}

function verificarAcesso() {
  try {
    let email = Session.getActiveUser().getEmail();
    if (!email || email === '') email = Session.getEffectiveUser().getEmail();
    return !!(email && email !== '');
  } catch (e) {
    return false;
  }
}

function getOrCreateSheet(baseName) {
  const ss  = SpreadsheetApp.getActiveSpreadsheet();
  const key = getUserKey();
  const name = baseName + '_' + key;
  return ss.getSheetByName(name) || ss.insertSheet(name);
}

function getSalario() {
  const sheet = getOrCreateSheet('Dados');
  return sheet.getRange('B1').getValue() || 0;
}

function setSalario(valor) {
  const sheet = getOrCreateSheet('Dados');
  sheet.getRange('A1').setValue('Salario');
  sheet.getRange('B1').setValue(valor);
  return { ok: true };
}

function getGastos() {
  const sheet = getOrCreateSheet('Gastos');
  const dados = sheet.getDataRange().getValues();
  if (dados.length <= 1) return [];
  return dados.slice(1).map((r, i) => ({
    id: i + 2,
    data: r[0] ? Utilities.formatDate(new Date(r[0]), 'America/Sao_Paulo', 'dd/MM/yyyy') : '',
    categoria: r[1],
    valor: r[2],
    descricao: r[3]
  })).reverse();
}

function addGasto(data, categoria, valor, descricao) {
  const sheet = getOrCreateSheet('Gastos');
  if (sheet.getLastRow() === 0)
    sheet.appendRow(['Data', 'Categoria', 'Valor', 'Descrição']);
  sheet.appendRow([new Date(data + 'T00:00:00'), categoria, parseFloat(valor), descricao]);
  return { ok: true };
}

function deleteGasto(rowIndex) {
  getOrCreateSheet('Gastos').deleteRow(rowIndex);
  return { ok: true };
}

function getInformativos() {
  const sheet = getOrCreateSheet('Informativos');
  const dados = sheet.getDataRange().getValues();
  if (dados.length <= 1) return [];
  return dados.slice(1).map((r, i) => ({
    id: i + 2,
    data: r[0] ? Utilities.formatDate(new Date(r[0]), 'America/Sao_Paulo', 'dd/MM/yyyy') : '',
    titulo: r[1],
    texto: r[2]
  })).reverse();
}

function addInformativo(data, titulo, texto) {
  const sheet = getOrCreateSheet('Informativos');
  if (sheet.getLastRow() === 0)
    sheet.appendRow(['Data', 'Título', 'Texto']);
  sheet.appendRow([new Date(data + 'T00:00:00'), titulo, texto]);
  return { ok: true };
}

function deleteInformativo(rowIndex) {
  getOrCreateSheet('Informativos').deleteRow(rowIndex);
  return { ok: true };
}

function getResumo() {
  const salario = getSalario();
  const gastos  = getGastos();
  const total   = gastos.reduce((s, g) => s + g.valor, 0);
  const saldo   = salario - total;
  const porCategoria = {};
  gastos.forEach(g => {
    porCategoria[g.categoria] = (porCategoria[g.categoria] || 0) + g.valor;
  });
  return { salario, total, saldo, porCategoria };
}

---INDEX.HTML

<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8"/>
  <meta name="viewport" content="width=device-width, initial-scale=1"/>
  <title>💰 FinanceApp - Nilson Bezerra</title>
  <?!= include('Estilo'); ?>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap" rel="stylesheet"/>
  <link href="https://fonts.googleapis.com/icon?family=Material+Icons" rel="stylesheet"/>
</head>
<body>

<!-- ══════════════ TELA DE LOGIN ══════════════ -->
<div id="tela-login" class="login-overlay">
  <div class="login-box">
    <div class="login-logo">
      <span class="material-icons">account_balance_wallet</span>
    </div>
    <h1 class="login-title">FinanceApp</h1>
    <p class="login-sub">Controle financeiro pessoal inteligente</p>

    <div id="login-loading" class="login-loading">
      <div class="spinner"></div>
      <span>Verificando autenticação...</span>
    </div>

    <div id="login-content" style="display:none; width:100%;">
      <div id="user-preview" class="user-preview" style="display:none;">
        <div class="user-preview-avatar" id="preview-avatar"></div>
        <div class="user-preview-info">
          <span id="preview-nome"></span>
          <span id="preview-email"></span>
        </div>
      </div>

      <button class="btn-google" id="btn-entrar" onclick="fazerLogin()">
        <svg width="20" height="20" viewBox="0 0 48 48">
          <path fill="#EA4335" d="M24 9.5c3.54 0 6.71 1.22 9.21 3.6l6.85-6.85C35.9 2.38 30.47 0 24 0 14.62 0 6.51 5.38 2.56 13.22l7.98 6.19C12.43 13.72 17.74 9.5 24 9.5z"/>
          <path fill="#4285F4" d="M46.98 24.55c0-1.57-.15-3.09-.38-4.55H24v9.02h12.94c-.58 2.96-2.26 5.48-4.78 7.18l7.73 6c4.51-4.18 7.09-10.36 7.09-17.65z"/>
          <path fill="#FBBC05" d="M10.53 28.59c-.48-1.45-.76-2.99-.76-4.59s.27-3.14.76-4.59l-7.98-6.19C.92 16.46 0 20.12 0 24c0 3.88.92 7.54 2.56 10.78l7.97-6.19z"/>
          <path fill="#34A853" d="M24 48c6.48 0 11.93-2.13 15.89-5.81l-7.73-6c-2.18 1.48-4.97 2.31-8.16 2.31-6.26 0-11.57-4.22-13.47-9.91l-7.98 6.19C6.51 42.62 14.62 48 24 48z"/>
        </svg>
        Entrar com Google
      </button>

      <p class="login-info">
        <span class="material-icons" style="font-size:14px;vertical-align:middle;">lock</span>
        Seus dados ficam salvos na sua própria planilha Google
      </p>
    </div>

    <div class="login-footer">
      Desenvolvido por <strong>Nilson Bezerra</strong>
      <span class="material-icons" style="font-size:13px;color:#4fc3f7;vertical-align:middle;">favorite</span>
    </div>
  </div>
</div>

<!-- ══════════════ APP PRINCIPAL ══════════════ -->
<div id="app" style="display:none;">

  <!-- SIDEBAR -->
  <div class="sidebar" id="sidebar">
    <div class="sidebar-header">
      <span class="material-icons logo-icon">account_balance_wallet</span>
      <span class="logo-text">FinanceApp</span>
    </div>

    <div class="sidebar-user" id="sidebar-user">
      <div class="sidebar-avatar" id="sidebar-avatar">?</div>
      <div class="sidebar-user-info">
        <span class="sidebar-nome"  id="sidebar-nome">Carregando...</span>
        <span class="sidebar-email" id="sidebar-email"></span>
      </div>
    </div>

    <nav>
      <a class="nav-item active" onclick="showPage('dashboard')"    id="nav-dashboard">
        <span class="material-icons">dashboard</span> Dashboard
      </a>
      <a class="nav-item" onclick="showPage('gastos')"              id="nav-gastos">
        <span class="material-icons">receipt_long</span> Gastos
      </a>
      <a class="nav-item" onclick="showPage('informativos')"        id="nav-informativos">
        <span class="material-icons">sticky_note_2</span> Informativos
      </a>
      <a class="nav-item" onclick="showPage('configuracoes')"       id="nav-configuracoes">
        <span class="material-icons">settings</span> Configurações
      </a>
    </nav>

    <div class="sidebar-footer">
      <a class="nav-item nav-logout" onclick="confirmarLogout()">
        <span class="material-icons">logout</span> Sair
      </a>
      <div class="sidebar-credit">
        Desenvolvido por<br/><strong>Nilson Bezerra</strong>
      </div>
    </div>
  </div>

  <!-- TOPBAR -->
  <div class="topbar">
    <button class="menu-btn" onclick="toggleSidebar()">
      <span class="material-icons">menu</span>
    </button>
    <span id="page-title" class="page-title">Dashboard</span>
    <div class="topbar-right">
      <div class="topbar-user" onclick="showPage('configuracoes')">
        <div class="avatar" id="topbar-avatar">F</div>
        <span class="topbar-nome" id="topbar-nome"></span>
      </div>
    </div>
  </div>

  <!-- CONTEÚDO -->
  <div class="main-content">

    <!-- DASHBOARD -->
    <div id="page-dashboard" class="page active">
      <div class="page-header">
        <h2>Dashboard</h2>
        <span class="subtitle" id="mes-atual"></span>
      </div>
      <div class="cards-grid">
        <div class="card card-green">
          <div class="card-icon"><span class="material-icons">payments</span></div>
          <div class="card-info">
            <span class="card-label">Salário</span>
            <span class="card-value" id="dash-salario">R$ 0,00</span>
          </div>
        </div>
        <div class="card card-red">
          <div class="card-icon"><span class="material-icons">trending_down</span></div>
          <div class="card-info">
            <span class="card-label">Total de Gastos</span>
            <span class="card-value" id="dash-gastos">R$ 0,00</span>
          </div>
        </div>
        <div class="card card-blue" id="card-saldo">
          <div class="card-icon"><span class="material-icons">account_balance</span></div>
          <div class="card-info">
            <span class="card-label">Saldo do Mês</span>
            <span class="card-value" id="dash-saldo">R$ 0,00</span>
          </div>
        </div>
      </div>

      <div class="progress-card">
        <div class="progress-header">
          <span>Orçamento utilizado</span>
          <span id="pct-label">0%</span>
        </div>
        <div class="progress-bar-bg">
          <div class="progress-bar-fill" id="progress-bar"></div>
        </div>
      </div>

      <div class="section-title">Gastos por Categoria</div>
      <div id="categorias-container" class="categorias-grid"></div>
    </div>

    <!-- GASTOS -->
    <div id="page-gastos" class="page">
      <div class="page-header">
        <h2>Gastos</h2>
        <button class="btn-primary" onclick="openModal('modal-gasto')">
          <span class="material-icons">add</span> Novo Gasto
        </button>
      </div>
      <div class="table-card">
        <table class="data-table">
          <thead>
            <tr><th>Data</th><th>Categoria</th><th>Descrição</th><th>Valor</th><th></th></tr>
          </thead>
          <tbody id="tabela-gastos">
            <tr><td colspan="5" class="loading">Carregando...</td></tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- INFORMATIVOS -->
    <div id="page-informativos" class="page">
      <div class="page-header">
        <h2>Informativos</h2>
        <button class="btn-primary" onclick="openModal('modal-info')">
          <span class="material-icons">add</span> Novo Informativo
        </button>
      </div>
      <div id="cards-informativos" class="info-grid"></div>
    </div>

    <!-- CONFIGURAÇÕES -->
    <div id="page-configuracoes" class="page">
      <div class="page-header"><h2>Configurações</h2></div>

      <div class="config-card" style="margin-bottom:20px;">
        <h3>👤 Meu Perfil</h3>
        <p>Informações da conta Google conectada.</p>
        <div class="perfil-info">
          <div class="perfil-avatar" id="perfil-avatar"></div>
          <div>
            <div class="perfil-nome"  id="perfil-nome"></div>
            <div class="perfil-email" id="perfil-email"></div>
          </div>
        </div>
      </div>

      <div class="config-card" style="margin-bottom:20px;">
        <h3>💵 Definir Salário</h3>
        <p>Atualize o valor do seu salário mensal.</p>
        <div class="input-group">
          <label>Salário (R$)</label>
          <input type="number" id="input-salario" placeholder="Ex: 5000.00" step="0.01"/>
        </div>
        <button class="btn-primary" onclick="salvarSalario()">
          <span class="material-icons">save</span> Salvar
        </button>
      </div>

      <div class="config-card credit-card">
        <div class="credit-icon">
          <span class="material-icons">account_balance_wallet</span>
        </div>
        <div class="credit-info">
          <span class="credit-label">Desenvolvido por</span>
          <span class="credit-name">Nilson Bezerra</span>
          <span class="credit-desc">FinanceApp © <?= new Date().getFullYear() ?></span>
        </div>
      </div>
    </div>

  </div><!-- /main-content -->

  <!-- RODAPÉ FIXO -->
  <div class="footer-app">
    <span class="material-icons footer-icon">account_balance_wallet</span>
    <span>FinanceApp — Desenvolvido por <strong>Nilson Bezerra</strong></span>
    <span class="material-icons footer-heart">favorite</span>
  </div>

</div><!-- /app -->

<!-- MODAL GASTO -->
<div class="modal-overlay" id="modal-gasto">
  <div class="modal">
    <div class="modal-header">
      <h3>➕ Novo Gasto</h3>
      <button class="close-btn" onclick="closeModal('modal-gasto')">
        <span class="material-icons">close</span>
      </button>
    </div>
    <div class="modal-body">
      <div class="input-group">
        <label>📅 Data</label>
        <input type="date" id="g-data"/>
      </div>
      <div class="input-group">
        <label>📂 Categoria</label>
        <select id="g-categoria">
          <option>🛒 Alimentação</option>
          <option>🏠 Moradia</option>
          <option>🚗 Transporte</option>
          <option>💊 Saúde</option>
          <option>📚 Educação</option>
          <option>🎭 Lazer</option>
          <option>👗 Vestuário</option>
          <option>💡 Contas/Serviços</option>
          <option>📦 Outros</option>
        </select>
      </div>
      <div class="input-group">
        <label>💲 Valor (R$)</label>
        <input type="number" id="g-valor" placeholder="0,00" step="0.01"/>
      </div>
      <div class="input-group">
        <label>📝 Descrição</label>
        <input type="text" id="g-descricao" placeholder="Ex: Mercado, Uber..."/>
      </div>
    </div>
    <div class="modal-footer">
      <button class="btn-secondary" onclick="closeModal('modal-gasto')">Cancelar</button>
      <button class="btn-primary" onclick="salvarGasto()">
        <span class="material-icons">save</span> Salvar
      </button>
    </div>
  </div>
</div>

<!-- MODAL INFORMATIVO -->
<div class="modal-overlay" id="modal-info">
  <div class="modal">
    <div class="modal-header">
      <h3>📌 Novo Informativo</h3>
      <button class="close-btn" onclick="closeModal('modal-info')">
        <span class="material-icons">close</span>
      </button>
    </div>
    <div class="modal-body">
      <div class="input-group">
        <label>📅 Data</label>
        <input type="date" id="i-data"/>
      </div>
      <div class="input-group">
        <label>🏷️ Título</label>
        <input type="text" id="i-titulo" placeholder="Título do informativo"/>
      </div>
      <div class="input-group">
        <label>📝 Texto</label>
        <textarea id="i-texto" rows="4" placeholder="Escreva sua observação..."></textarea>
      </div>
    </div>
    <div class="modal-footer">
      <button class="btn-secondary" onclick="closeModal('modal-info')">Cancelar</button>
      <button class="btn-primary" onclick="salvarInformativo()">
        <span class="material-icons">save</span> Salvar
      </button>
    </div>
  </div>
</div>

<!-- MODAL LOGOUT -->
<div class="modal-overlay" id="modal-logout">
  <div class="modal" style="max-width:360px;">
    <div class="modal-header">
      <h3>👋 Sair da conta</h3>
      <button class="close-btn" onclick="closeModal('modal-logout')">
        <span class="material-icons">close</span>
      </button>
    </div>
    <div class="modal-body">
      <p style="color:#555;font-size:14px;line-height:1.6;">
        Deseja sair da sua conta? Seus dados continuarão salvos na planilha.
      </p>
    </div>
    <div class="modal-footer">
      <button class="btn-secondary" onclick="closeModal('modal-logout')">Cancelar</button>
      <button class="btn-danger" onclick="fazerLogout()">
        <span class="material-icons">logout</span> Sair
      </button>
    </div>
  </div>
</div>

<!-- MODAL DE AUTORIZAÇÃO -->
<div class="modal-overlay" id="modal-auth">
  <div class="modal" style="max-width:400px;">
    <div class="modal-header">
      <h3>🔐 Autorização necessária</h3>
      <button class="close-btn" onclick="closeModal('modal-auth')">
        <span class="material-icons">close</span>
      </button>
    </div>
    <div class="modal-body">
      <p style="color:#555;font-size:14px;line-height:1.6;">
        Para acessar o app, você precisa autorizar o acesso à sua conta Google.
        Clique no botão abaixo, autorize e depois recarregue esta página.
      </p>
    </div>
    <div class="modal-footer">
      <button class="btn-secondary" onclick="closeModal('modal-auth')">Cancelar</button>
      <button class="btn-primary" id="btn-autorizar" onclick="abrirAutorizacao()">
        <span class="material-icons">open_in_new</span> Autorizar acesso
      </button>
    </div>
  </div>
</div>

<!-- TOAST -->
<div id="toast" class="toast"></div>

<script>
  let usuarioAtual  = null;
  let authUrlGlobal = null;

  document.addEventListener('DOMContentLoaded', () => {
    const hoje = new Date().toISOString().split('T')[0];
    document.getElementById('g-data').value = hoje;
    document.getElementById('i-data').value = hoje;
    const meses = ['Janeiro','Fevereiro','Março','Abril','Maio','Junho',
                   'Julho','Agosto','Setembro','Outubro','Novembro','Dezembro'];
    const now = new Date();
    document.getElementById('mes-atual').textContent =
      `${meses[now.getMonth()]} ${now.getFullYear()}`;
    verificarLogin();
  });

  // ── LOGIN ────────────────────────────────────────────
  function verificarLogin() {
    const loginLoading = document.getElementById('login-loading');
    const loginContent = document.getElementById('login-content');

    google.script.run
      .withSuccessHandler(user => {
        loginLoading.style.display = 'none';
        loginContent.style.display = 'flex';
        loginContent.style.flexDirection = 'column';
        loginContent.style.alignItems    = 'center';

        if (user && user.email) {
          usuarioAtual = user;
          document.getElementById('user-preview').style.display = 'flex';
          document.getElementById('preview-avatar').textContent = user.avatar;
          document.getElementById('preview-nome').textContent   = user.nome;
          document.getElementById('preview-email').textContent  = user.email;
          document.getElementById('btn-entrar').innerHTML =
            `<span class="material-icons">login</span> Continuar como ${user.nome.split(' ')[0]}`;
          // Entra direto se já autenticado
          iniciarApp(user);
        } else {
          // Busca URL de autorização em paralelo
          google.script.run
            .withSuccessHandler(url => { authUrlGlobal = url; })
            .withFailureHandler(() => {})
            .getAuthUrl();
          document.getElementById('btn-entrar').innerHTML =
            svgGoogle() + ' Entrar com Google';
        }
      })
      .withFailureHandler(err => {
        loginLoading.style.display = 'none';
        loginContent.style.display = 'flex';
        loginContent.style.flexDirection = 'column';
        loginContent.style.alignItems    = 'center';
        document.getElementById('btn-entrar').innerHTML =
          svgGoogle() + ' Entrar com Google';
        showToast('❌ Erro de conexão: ' + err.message, 'warn');
      })
      .getUsuarioLogado();
  }

  function fazerLogin() {
    const btn = document.getElementById('btn-entrar');
    btn.disabled = true;
    btn.innerHTML = '<div class="spinner-small"></div> Verificando...';

    google.script.run
      .withSuccessHandler(user => {
        if (user && user.email) {
          usuarioAtual = user;
          iniciarApp(user);
        } else {
          btn.disabled = false;
          btn.innerHTML = svgGoogle() + ' Entrar com Google';
          // Tenta obter a URL de autorização
          google.script.run
            .withSuccessHandler(url => {
              if (url) {
                authUrlGlobal = url;
                openModal('modal-auth');
              } else {
                showToast('⚠️ Não foi possível identificar o usuário. Tente recarregar a página.', 'warn');
              }
            })
            .withFailureHandler(() => {
              showToast('⚠️ Erro ao obter autorização. Tente recarregar a página.', 'warn');
            })
            .getAuthUrl();
        }
      })
      .withFailureHandler(err => {
        btn.disabled = false;
        btn.innerHTML = svgGoogle() + ' Entrar com Google';
        showToast('❌ Erro ao autenticar: ' + err.message, 'warn');
      })
      .getUsuarioLogado();
  }

  function abrirAutorizacao() {
    if (authUrlGlobal) {
      window.open(authUrlGlobal, '_blank');
      closeModal('modal-auth');
      showToast('✅ Após autorizar, recarregue a página.', 'info');
    } else {
      showToast('⚠️ URL de autorização não disponível. Recarregue a página.', 'warn');
    }
  }

  function svgGoogle() {
    return `<svg width="20" height="20" viewBox="0 0 48 48">
      <path fill="#EA4335" d="M24 9.5c3.54 0 6.71 1.22 9.21 3.6l6.85-6.85C35.9 2.38 30.47 0 24 0 14.62 0 6.51 5.38 2.56 13.22l7.98 6.19C12.43 13.72 17.74 9.5 24 9.5z"/>
      <path fill="#4285F4" d="M46.98 24.55c0-1.57-.15-3.09-.38-4.55H24v9.02h12.94c-.58 2.96-2.26 5.48-4.78 7.18l7.73 6c4.51-4.18 7.09-10.36 7.09-17.65z"/>
      <path fill="#FBBC05" d="M10.53 28.59c-.48-1.45-.76-2.99-.76-4.59s.27-3.14.76-4.59l-7.98-6.19C.92 16.46 0 20.12 0 24c0 3.88.92 7.54 2.56 10.78l7.97-6.19z"/>
      <path fill="#34A853" d="M24 48c6.48 0 11.93-2.13 15.89-5.81l-7.73-6c-2.18 1.48-4.97 2.31-8.16 2.31-6.26 0-11.57-4.22-13.47-9.91l-7.98 6.19C6.51 42.62 14.62 48 24 48z"/>
    </svg>`;
  }

  function iniciarApp(user) {
    document.getElementById('tela-login').style.display = 'none';
    document.getElementById('app').style.display        = 'flex';
    document.getElementById('sidebar-avatar').textContent = user.avatar;
    document.getElementById('sidebar-nome').textContent   = user.nome;
    document.getElementById('sidebar-email').textContent  = user.email;
    document.getElementById('topbar-avatar').textContent  = user.avatar;
    document.getElementById('topbar-nome').textContent    = user.nome.split(' ')[0];
    document.getElementById('perfil-avatar').textContent  = user.avatar;
    document.getElementById('perfil-nome').textContent    = user.nome;
    document.getElementById('perfil-email').textContent   = user.email;
    carregarDashboard();
    carregarGastos();
    carregarInformativos();
    carregarSalarioConfig();
  }

  function confirmarLogout() { openModal('modal-logout'); }

  function fazerLogout() {
    closeModal('modal-logout');
    usuarioAtual  = null;
    authUrlGlobal = null;
    document.getElementById('app').style.display           = 'none';
    document.getElementById('tela-login').style.display    = 'flex';
    document.getElementById('user-preview').style.display  = 'none';
    document.getElementById('login-loading').style.display = 'none';
    document.getElementById('login-content').style.display = 'flex';
    const btn = document.getElementById('btn-entrar');
    btn.disabled  = false;
    btn.innerHTML = svgGoogle() + ' Entrar com Google';
    showToast('👋 Até logo!', 'info');
  }

  // ── NAVEGAÇÃO ────────────────────────────────────────
  function showPage(page) {
    document.querySelectorAll('.page').forEach(p => p.classList.remove('active'));
    document.querySelectorAll('.nav-item').forEach(n => n.classList.remove('active'));
    document.getElementById('page-' + page).classList.add('active');
    document.getElementById('nav-'  + page).classList.add('active');
    const titles = { dashboard:'Dashboard', gastos:'Gastos',
                     informativos:'Informativos', configuracoes:'Configurações' };
    document.getElementById('page-title').textContent = titles[page];
    if (window.innerWidth < 768) document.getElementById('sidebar').classList.remove('open');
  }

  function toggleSidebar() {
    document.getElementById('sidebar').classList.toggle('open');
  }

  // ── DASHBOARD ────────────────────────────────────────
  function carregarDashboard() {
    google.script.run
      .withSuccessHandler(res => {
        document.getElementById('dash-salario').textContent = fmt(res.salario);
        document.getElementById('dash-gastos').textContent  = fmt(res.total);
        document.getElementById('dash-saldo').textContent   = fmt(res.saldo);
        const pct = res.salario > 0 ? Math.min((res.total / res.salario) * 100, 100) : 0;
        document.getElementById('pct-label').textContent = pct.toFixed(1) + '%';
        const bar = document.getElementById('progress-bar');
        bar.style.width      = pct + '%';
        bar.style.background = pct > 90 ? '#ef5350' : pct > 70 ? '#ffa726' : '#66bb6a';
        document.getElementById('card-saldo').className =
          'card ' + (res.saldo >= 0 ? 'card-blue' : 'card-red');
        const cont = document.getElementById('categorias-container');
        cont.innerHTML = '';
        const cores = {
          '🛒 Alimentação':'#ef9a9a','🏠 Moradia':'#90caf9','🚗 Transporte':'#a5d6a7',
          '💊 Saúde':'#f48fb1','📚 Educação':'#ffe082','🎭 Lazer':'#ce93d8',
          '👗 Vestuário':'#80cbc4','💡 Contas/Serviços':'#ffcc80','📦 Outros':'#b0bec5'
        };
        for (const [cat, val] of Object.entries(res.porCategoria)) {
          const pctCat = res.total > 0 ? ((val / res.total) * 100).toFixed(1) : 0;
          const div = document.createElement('div');
          div.className = 'cat-card';
          div.innerHTML = `
            <div class="cat-top">
              <span class="cat-name">${cat}</span>
              <span class="cat-pct">${pctCat}%</span>
            </div>
            <div class="cat-val">${fmt(val)}</div>
            <div class="cat-bar-bg">
              <div class="cat-bar-fill" style="width:${pctCat}%;background:${cores[cat]||'#90caf9'}"></div>
            </div>`;
          cont.appendChild(div);
        }
        if (!Object.keys(res.porCategoria).length)
          cont.innerHTML = '<p class="empty-msg">Nenhum gasto registrado ainda.</p>';
      })
      .withFailureHandler(err => {
        showToast('❌ Erro ao carregar dashboard: ' + err.message, 'warn');
      })
      .getResumo();
  }

  // ── GASTOS ───────────────────────────────────────────
  function carregarGastos() {
    google.script.run
      .withSuccessHandler(lista => {
        const tbody = document.getElementById('tabela-gastos');
        if (!lista.length) {
          tbody.innerHTML = '<tr><td colspan="5" class="empty-msg">Nenhum gasto encontrado.</td></tr>';
          return;
        }
        tbody.innerHTML = lista.map(g => `
          <tr>
            <td>${g.data}</td>
            <td><span class="badge">${g.categoria}</span></td>
            <td>${g.descricao}</td>
            <td class="valor-neg">${fmt(g.valor)}</td>
            <td><button class="btn-icon" onclick="excluirGasto(${g.id})">
              <span class="material-icons">delete_outline</span></button></td>
          </tr>`).join('');
      })
      .withFailureHandler(err => {
        showToast('❌ Erro ao carregar gastos: ' + err.message, 'warn');
      })
      .getGastos();
  }

  function salvarGasto() {
    const data = document.getElementById('g-data').value;
    const cat  = document.getElementById('g-categoria').value;
    const val  = document.getElementById('g-valor').value;
    const desc = document.getElementById('g-descricao').value;
    if (!data || !val) { showToast('⚠️ Preencha data e valor!', 'warn'); return; }
    showToast('Salvando...', 'info');
    google.script.run
      .withSuccessHandler(() => {
        closeModal('modal-gasto');
        carregarGastos();
        carregarDashboard();
        showToast('✅ Gasto adicionado!', 'success');
      })
      .withFailureHandler(err => {
        showToast('❌ Erro ao salvar gasto: ' + err.message, 'warn');
      })
      .addGasto(data, cat, val, desc);
  }

  function excluirGasto(id) {
    if (!confirm('Excluir este gasto?')) return;
    google.script.run
      .withSuccessHandler(() => {
        carregarGastos();
        carregarDashboard();
        showToast('🗑️ Gasto removido!', 'success');
      })
      .withFailureHandler(err => {
        showToast('❌ Erro ao excluir: ' + err.message, 'warn');
      })
      .deleteGasto(id);
  }

  // ── INFORMATIVOS ─────────────────────────────────────
  function carregarInformativos() {
    google.script.run
      .withSuccessHandler(lista => {
        const cont = document.getElementById('cards-informativos');
        if (!lista.length) {
          cont.innerHTML = '<p class="empty-msg">Nenhum informativo registrado.</p>';
          return;
        }
        cont.innerHTML = lista.map(i => `
          <div class="info-card">
            <div class="info-card-header">
              <span class="info-titulo">${i.titulo}</span>
              <button class="btn-icon" onclick="excluirInfo(${i.id})">
                <span class="material-icons">delete_outline</span></button>
            </div>
            <p class="info-texto">${i.texto}</p>
            <span class="info-data">${i.data}</span>
          </div>`).join('');
      })
      .withFailureHandler(err => {
        showToast('❌ Erro ao carregar informativos: ' + err.message, 'warn');
      })
      .getInformativos();
  }

  function salvarInformativo() {
    const data  = document.getElementById('i-data').value;
    const tit   = document.getElementById('i-titulo').value;
    const texto = document.getElementById('i-texto').value;
    if (!tit || !texto) { showToast('⚠️ Preencha título e texto!', 'warn'); return; }
    google.script.run
      .withSuccessHandler(() => {
        closeModal('modal-info');
        carregarInformativos();
        showToast('✅ Informativo salvo!', 'success');
      })
      .withFailureHandler(err => {
        showToast('❌ Erro ao salvar informativo: ' + err.message, 'warn');
      })
      .addInformativo(data, tit, texto);
  }

  function excluirInfo(id) {
    if (!confirm('Excluir este informativo?')) return;
    google.script.run
      .withSuccessHandler(() => {
        carregarInformativos();
        showToast('🗑️ Informativo removido!', 'success');
      })
      .withFailureHandler(err => {
        showToast('❌ Erro ao excluir: ' + err.message, 'warn');
      })
      .deleteInformativo(id);
  }

  // ── CONFIGURAÇÕES ────────────────────────────────────
  function carregarSalarioConfig() {
    google.script.run
      .withSuccessHandler(v => {
        document.getElementById('input-salario').value = v || '';
      })
      .withFailureHandler(() => {})
      .getSalario();
  }

  function salvarSalario() {
    const val = document.getElementById('input-salario').value;
    if (!val) { showToast('⚠️ Informe o salário!', 'warn'); return; }
    google.script.run
      .withSuccessHandler(() => {
        carregarDashboard();
        showToast('✅ Salário atualizado!', 'success');
      })
      .withFailureHandler(err => {
        showToast('❌ Erro ao salvar salário: ' + err.message, 'warn');
      })
      .setSalario(parseFloat(val));
  }

  // ── HELPERS ──────────────────────────────────────────
  function fmt(v) {
    return (v || 0).toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' });
  }
  function openModal(id)  { document.getElementById(id).classList.add('active'); }
  function closeModal(id) { document.getElementById(id).classList.remove('active'); }
  function showToast(msg, tipo) {
    const t = document.getElementById('toast');
    t.textContent = msg;
    t.className   = 'toast show ' + (tipo || '');
    setTimeout(() => t.className = 'toast', 3000);
  }
  document.querySelectorAll('.modal-overlay').forEach(m => {
    m.addEventListener('click', e => { if (e.target === m) m.classList.remove('active'); });
  });
</script>
</body>
</html>

--ESTILO.HTML

<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
body {
  font-family: 'Inter', sans-serif;
  background: #f0f2f5;
  color: #1a1a2e;
  display: flex;
  min-height: 100vh;
}

/* ── LOGIN ── */
.login-overlay {
  position: fixed; inset: 0; z-index: 999;
  background: linear-gradient(135deg, #0f0c29, #302b63, #24243e);
  display: flex; align-items: center; justify-content: center;
}
.login-box {
  background: rgba(255,255,255,.07);
  backdrop-filter: blur(20px);
  border: 1px solid rgba(255,255,255,.12);
  border-radius: 24px;
  padding: 48px 40px 32px;
  width: 100%; max-width: 400px;
  display: flex; flex-direction: column; align-items: center;
  gap: 16px;
  box-shadow: 0 30px 80px rgba(0,0,0,.5);
}
.login-logo {
  width: 72px; height: 72px; border-radius: 20px;
  background: linear-gradient(135deg, #4fc3f7, #1565c0);
  display: flex; align-items: center; justify-content: center;
}
.login-logo .material-icons { font-size: 36px; color: #fff; }
.login-title { font-size: 28px; font-weight: 700; color: #fff; }
.login-sub   { font-size: 14px; color: rgba(255,255,255,.5); text-align: center; }
.login-loading {
  display: flex; align-items: center; gap: 12px;
  color: rgba(255,255,255,.6); font-size: 14px; padding: 16px 0;
}
.user-preview {
  display: flex; align-items: center; gap: 12px;
  background: rgba(255,255,255,.1); border-radius: 14px;
  padding: 14px 18px; width: 100%;
}
.user-preview-avatar {
  width: 44px; height: 44px; border-radius: 50%;
  background: linear-gradient(135deg, #4fc3f7, #1565c0);
  color: #fff; font-size: 18px; font-weight: 700;
  display: flex; align-items: center; justify-content: center; flex-shrink: 0;
}
.user-preview-info { display: flex; flex-direction: column; gap: 2px; overflow: hidden; }
.user-preview-info span:first-child { font-size: 14px; font-weight: 600; color: #fff; }
.user-preview-info span:last-child  { font-size: 12px; color: rgba(255,255,255,.5); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.btn-google {
  display: flex; align-items: center; justify-content: center; gap: 10px;
  width: 100%; padding: 14px; border-radius: 12px;
  background: #fff; color: #333; border: none;
  font-size: 15px; font-weight: 600; cursor: pointer;
  transition: all .2s; box-shadow: 0 4px 15px rgba(0,0,0,.2);
}
.btn-google:hover    { transform: translateY(-2px); box-shadow: 0 8px 25px rgba(0,0,0,.3); }
.btn-google:disabled { opacity: .7; cursor: not-allowed; transform: none; }
.login-info {
  font-size: 12px; color: rgba(255,255,255,.35); text-align: center; line-height: 1.5;
}
.login-footer {
  margin-top: 8px;
  font-size: 12px; color: rgba(255,255,255,.3);
  text-align: center; display: flex; align-items: center; gap: 5px;
  border-top: 1px solid rgba(255,255,255,.08);
  padding-top: 16px; width: 100%; justify-content: center;
}
.login-footer strong { color: rgba(255,255,255,.6); }

/* ── SPINNER ── */
.spinner {
  width: 24px; height: 24px; border-radius: 50%;
  border: 3px solid rgba(255,255,255,.2);
  border-top-color: #4fc3f7;
  animation: spin .8s linear infinite;
}
.spinner-small {
  width: 16px; height: 16px; border-radius: 50%;
  border: 2px solid rgba(0,0,0,.15); border-top-color: #333;
  animation: spin .8s linear infinite; display: inline-block;
}
@keyframes spin { to { transform: rotate(360deg); } }

/* ── SIDEBAR ── */
.sidebar {
  width: 240px; min-height: 100vh;
  background: linear-gradient(160deg, #1a1a2e 0%, #16213e 100%);
  display: flex; flex-direction: column;
  position: fixed; left: 0; top: 0; z-index: 100;
  transition: transform .3s ease;
}
.sidebar-header {
  display: flex; align-items: center; gap: 10px;
  padding: 22px 20px; border-bottom: 1px solid rgba(255,255,255,.07);
}
.logo-icon { color: #4fc3f7; font-size: 26px; }
.logo-text  { color: #fff; font-size: 17px; font-weight: 700; }
.sidebar-user {
  display: flex; align-items: center; gap: 10px;
  padding: 14px 20px; background: rgba(255,255,255,.04);
  border-bottom: 1px solid rgba(255,255,255,.07);
}
.sidebar-avatar {
  width: 38px; height: 38px; border-radius: 50%;
  background: linear-gradient(135deg, #4fc3f7, #1565c0);
  color: #fff; font-size: 15px; font-weight: 700;
  display: flex; align-items: center; justify-content: center; flex-shrink: 0;
}
.sidebar-user-info { display: flex; flex-direction: column; gap: 2px; overflow: hidden; }
.sidebar-nome  { font-size: 13px; font-weight: 600; color: #fff; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.sidebar-email { font-size: 11px; color: rgba(255,255,255,.4); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
nav { padding: 12px 0; flex: 1; }
.nav-item {
  display: flex; align-items: center; gap: 12px;
  padding: 13px 20px; color: rgba(255,255,255,.55);
  cursor: pointer; font-size: 14px; font-weight: 500;
  border-left: 3px solid transparent;
  transition: all .2s; text-decoration: none;
}
.nav-item:hover  { background: rgba(255,255,255,.06); color: #fff; }
.nav-item.active { background: rgba(79,195,247,.12); color: #4fc3f7; border-left-color: #4fc3f7; }
.nav-item .material-icons { font-size: 20px; }
.sidebar-footer { border-top: 1px solid rgba(255,255,255,.07); padding: 8px 0 0; }
.nav-logout { color: rgba(255,100,100,.6) !important; }
.nav-logout:hover { background: rgba(239,83,80,.1) !important; color: #ef5350 !important; }
.sidebar-credit {
  padding: 12px 20px 16px;
  font-size: 11px; color: rgba(255,255,255,.2);
  line-height: 1.6; text-align: center;
  border-top: 1px solid rgba(255,255,255,.05);
  margin-top: 4px;
}
.sidebar-credit strong { color: rgba(255,255,255,.45); }

/* ── TOPBAR ── */
.topbar {
  position: fixed; top: 0; left: 240px; right: 0; height: 60px;
  background: #fff; display: flex; align-items: center;
  padding: 0 24px; gap: 16px;
  box-shadow: 0 1px 8px rgba(0,0,0,.07); z-index: 99;
}
.menu-btn { display: none; background: none; border: none; cursor: pointer; color: #555; }
.page-title { font-size: 17px; font-weight: 600; flex: 1; }
.topbar-right { display: flex; align-items: center; gap: 10px; }
.topbar-user {
  display: flex; align-items: center; gap: 8px;
  cursor: pointer; padding: 6px 10px; border-radius: 10px; transition: background .2s;
}
.topbar-user:hover { background: #f5f5f5; }
.topbar-nome { font-size: 13px; font-weight: 600; color: #333; }
.avatar {
  width: 34px; height: 34px; border-radius: 50%;
  background: linear-gradient(135deg, #4fc3f7, #1565c0);
  color: #fff; font-weight: 700;
  display: flex; align-items: center; justify-content: center; font-size: 14px;
}

/* ── MAIN ── */
.main-content {
  margin-left: 240px; margin-top: 60px;
  padding: 28px 28px 60px;
  flex: 1; min-height: calc(100vh - 60px);
}
.page { display: none; }
.page.active { display: block; animation: fadeIn .3s ease; }
@keyframes fadeIn { from { opacity:0; transform:translateY(10px); } to { opacity:1; transform:none; } }
.page-header {
  display: flex; align-items: center; justify-content: space-between; margin-bottom: 24px;
}
.page-header h2 { font-size: 22px; font-weight: 700; }
.subtitle { font-size: 13px; color: #888; margin-left: 10px; }

/* ── CARDS ── */
.cards-grid {
  display: grid; grid-template-columns: repeat(auto-fit, minmax(210px, 1fr));
  gap: 18px; margin-bottom: 24px;
}
.card {
  border-radius: 16px; padding: 22px;
  display: flex; align-items: center; gap: 16px;
  box-shadow: 0 2px 12px rgba(0,0,0,.07);
  transition: transform .2s, box-shadow .2s;
}
.card:hover { transform: translateY(-3px); box-shadow: 0 6px 20px rgba(0,0,0,.12); }
.card-green { background: linear-gradient(135deg,#e8f5e9,#c8e6c9); }
.card-red   { background: linear-gradient(135deg,#fce4ec,#ffcdd2); }
.card-blue  { background: linear-gradient(135deg,#e3f2fd,#bbdefb); }
.card-icon .material-icons { font-size: 36px; }
.card-green .card-icon .material-icons { color: #2e7d32; }
.card-red   .card-icon .material-icons { color: #c62828; }
.card-blue  .card-icon .material-icons { color: #1565c0; }
.card-info { display: flex; flex-direction: column; gap: 4px; }
.card-label { font-size: 12px; font-weight: 500; color: #555; text-transform: uppercase; letter-spacing: .5px; }
.card-value { font-size: 22px; font-weight: 700; color: #1a1a2e; }

/* ── PROGRESS ── */
.progress-card {
  background: #fff; border-radius: 16px; padding: 20px 24px;
  box-shadow: 0 2px 12px rgba(0,0,0,.07); margin-bottom: 24px;
}
.progress-header { display: flex; justify-content: space-between; font-size: 14px; font-weight: 600; margin-bottom: 10px; color: #444; }
.progress-bar-bg { background: #e0e0e0; border-radius: 999px; height: 12px; overflow: hidden; }
.progress-bar-fill { height: 100%; border-radius: 999px; transition: width .6s ease, background .4s; }

/* ── CATEGORIAS ── */
.section-title { font-size: 15px; font-weight: 700; margin-bottom: 14px; color: #333; }
.categorias-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(190px,1fr)); gap: 14px; }
.cat-card { background: #fff; border-radius: 14px; padding: 16px; box-shadow: 0 2px 10px rgba(0,0,0,.06); }
.cat-top { display: flex; justify-content: space-between; margin-bottom: 6px; }
.cat-name { font-size: 13px; font-weight: 600; color: #333; }
.cat-pct  { font-size: 12px; color: #888; }
.cat-val  { font-size: 16px; font-weight: 700; color: #1a1a2e; margin-bottom: 8px; }
.cat-bar-bg { background: #eee; border-radius: 999px; height: 6px; overflow: hidden; }
.cat-bar-fill { height: 100%; border-radius: 999px; transition: width .5s; }

/* ── TABELA ── */
.table-card { background: #fff; border-radius: 16px; box-shadow: 0 2px 12px rgba(0,0,0,.07); overflow: hidden; }
.data-table { width: 100%; border-collapse: collapse; }
.data-table thead tr { background: #f8f9fa; }
.data-table th { padding: 14px 16px; text-align: left; font-size: 12px; font-weight: 600; color: #888; text-transform: uppercase; letter-spacing: .5px; }
.data-table td { padding: 13px 16px; border-bottom: 1px solid #f1f1f1; font-size: 14px; }
.data-table tr:last-child td { border-bottom: none; }
.data-table tr:hover td { background: #fafafa; }
.badge { background: #e3f2fd; color: #1565c0; padding: 4px 10px; border-radius: 999px; font-size: 12px; font-weight: 500; }
.valor-neg { color: #c62828; font-weight: 600; }

/* ── INFORMATIVOS ── */
.info-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px,1fr)); gap: 18px; }
.info-card { background: #fff; border-radius: 16px; padding: 20px; box-shadow: 0 2px 12px rgba(0,0,0,.07); border-left: 4px solid #7c4dff; }
.info-card-header { display: flex; justify-content: space-between; align-items: flex-start; margin-bottom: 10px; }
.info-titulo { font-size: 15px; font-weight: 700; color: #1a1a2e; }
.info-texto  { font-size: 13px; color: #555; line-height: 1.6; margin-bottom: 12px; }
.info-data   { font-size: 11px; color: #aaa; }

/* ── CONFIGURAÇÕES ── */
.config-card { background: #fff; border-radius: 16px; padding: 28px; max-width: 480px; box-shadow: 0 2px 12px rgba(0,0,0,.07); }
.config-card h3 { font-size: 17px; font-weight: 700; margin-bottom: 6px; }
.config-card p  { color: #888; font-size: 13px; margin-bottom: 20px; }
.perfil-info { display: flex; align-items: center; gap: 16px; }
.perfil-avatar {
  width: 56px; height: 56px; border-radius: 50%;
  background: linear-gradient(135deg, #4fc3f7, #1565c0);
  color: #fff; font-size: 22px; font-weight: 700;
  display: flex; align-items: center; justify-content: center; flex-shrink: 0;
}
.perfil-nome  { font-size: 16px; font-weight: 700; color: #1a1a2e; }
.perfil-email { font-size: 13px; color: #888; margin-top: 2px; }

/* ── CARD CRÉDITO (configurações) ── */
.credit-card {
  display: flex !important; align-items: center; gap: 18px;
  background: linear-gradient(135deg, #1a1a2e, #16213e) !important;
  max-width: 480px;
}
.credit-icon {
  width: 52px; height: 52px; border-radius: 14px;
  background: linear-gradient(135deg, #4fc3f7, #1565c0);
  display: flex; align-items: center; justify-content: center; flex-shrink: 0;
}
.credit-icon .material-icons { font-size: 26px; color: #fff; }
.credit-info { display: flex; flex-direction: column; gap: 3px; }
.credit-label { font-size: 11px; color: rgba(255,255,255,.4); text-transform: uppercase; letter-spacing: .5px; }
.credit-name  { font-size: 18px; font-weight: 700; color: #fff; }
.credit-desc  { font-size: 12px; color: rgba(255,255,255,.3); }

/* ── INPUTS ── */
.input-group { display: flex; flex-direction: column; gap: 6px; margin-bottom: 16px; }
.input-group label { font-size: 13px; font-weight: 600; color: #444; }
.input-group input, .input-group select, .input-group textarea {
  padding: 11px 14px; border: 1.5px solid #e0e0e0;
  border-radius: 10px; font-size: 14px; font-family: 'Inter', sans-serif;
  outline: none; transition: border .2s; background: #fafafa; color: #1a1a2e;
}
.input-group input:focus, .input-group select:focus, .input-group textarea:focus {
  border-color: #4fc3f7; background: #fff;
}

/* ── BOTÕES ── */
.btn-primary {
  display: inline-flex; align-items: center; gap: 6px;
  background: linear-gradient(135deg, #1565c0, #4fc3f7);
  color: #fff; border: none; padding: 11px 20px;
  border-radius: 10px; font-size: 14px; font-weight: 600;
  cursor: pointer; transition: opacity .2s, transform .1s;
}
.btn-primary:hover { opacity: .9; transform: translateY(-1px); }
.btn-primary .material-icons { font-size: 18px; }
.btn-secondary {
  background: #f1f1f1; color: #333; border: none;
  padding: 11px 20px; border-radius: 10px; font-size: 14px; font-weight: 600;
  cursor: pointer; transition: background .2s;
}
.btn-secondary:hover { background: #e0e0e0; }
.btn-danger {
  display: inline-flex; align-items: center; gap: 6px;
  background: linear-gradient(135deg, #c62828, #ef5350);
  color: #fff; border: none; padding: 11px 20px;
  border-radius: 10px; font-size: 14px; font-weight: 600;
  cursor: pointer; transition: opacity .2s;
}
.btn-danger:hover { opacity: .9; }
.btn-danger .material-icons { font-size: 18px; }
.btn-icon {
  background: none; border: none; cursor: pointer; color: #aaa;
  display: flex; align-items: center; border-radius: 8px; padding: 4px;
  transition: color .2s, background .2s;
}
.btn-icon:hover { color: #ef5350; background: #fce4ec; }

/* ── MODAL ── */
.modal-overlay {
  display: none; position: fixed; inset: 0;
  background: rgba(0,0,0,.45); backdrop-filter: blur(4px);
  z-index: 200; align-items: center; justify-content: center;
}
.modal-overlay.active { display: flex; }
.modal {
  background: #fff; border-radius: 20px; width: 100%; max-width: 460px; margin: 16px;
  box-shadow: 0 20px 60px rgba(0,0,0,.2); animation: slideUp .3s ease;
}
@keyframes slideUp { from { opacity:0; transform:translateY(30px); } to { opacity:1; transform:none; } }
.modal-header { display: flex; justify-content: space-between; align-items: center; padding: 20px 24px; border-bottom: 1px solid #f1f1f1; }
.modal-header h3 { font-size: 17px; font-weight: 700; }
.close-btn { background: none; border: none; cursor: pointer; color: #aaa; }
.close-btn:hover { color: #333; }
.modal-body   { padding: 24px; }
.modal-footer { display: flex; justify-content: flex-end; gap: 10px; padding: 16px 24px; border-top: 1px solid #f1f1f1; }

/* ── RODAPÉ FIXO ── */
.footer-app {
  position: fixed; bottom: 0; left: 240px; right: 0; height: 38px;
  background: #fff; border-top: 1px solid #f0f0f0;
  display: flex; align-items: center; justify-content: center;
  gap: 6px; font-size: 12px; color: #bbb; z-index: 98;
}
.footer-app strong { color: #1565c0; font-weight: 600; }
.footer-icon  { font-size: 14px !important; color: #4fc3f7; }
.footer-heart { font-size: 13px !important; color: #ef5350; }

/* ── TOAST ── */
.toast {
  position: fixed; bottom: 52px; right: 28px;
  background: #1a1a2e; color: #fff; padding: 12px 20px;
  border-radius: 12px; font-size: 14px; font-weight: 500;
  opacity: 0; transform: translateY(10px);
  transition: all .3s; pointer-events: none; z-index: 999;
}
.toast.show    { opacity: 1; transform: none; }
.toast.success { background: #2e7d32; }
.toast.warn    { background: #e65100; }
.toast.info    { background: #1565c0; }

/* ── MISC ── */
.loading, .empty-msg { text-align: center; color: #aaa; padding: 30px; font-size: 14px; }

/* ── RESPONSIVE ── */
@media (max-width: 768px) {
  .sidebar { transform: translateX(-100%); }
  .sidebar.open { transform: translateX(0); }
  .topbar { left: 0; }
  .main-content { margin-left: 0; padding: 20px 16px 60px; }
  .menu-btn { display: flex; }
  .cards-grid { grid-template-columns: 1fr 1fr; }
  .topbar-nome { display: none; }
  .footer-app { left: 0; }
}
@media (max-width: 480px) {
  .cards-grid { grid-template-columns: 1fr; }
  .login-box { padding: 36px 24px 24px; }
}
</style>
