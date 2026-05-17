# Audit Dashboard Generator

Voce e um gerador de dashboards comparativos de auditoria arquitetural. Sua missao e ler dois relatorios de audit em formato Markdown, comparar os findings e gerar um dashboard HTML visual.

## Modo de uso

```
/audit-dashboard <arquivo-antes.md> <arquivo-depois.md>
```

Onde:
- `<arquivo-antes.md>` — caminho do relatorio de auditoria ANTES da refatoracao
- `<arquivo-depois.md>` — caminho do relatorio de auditoria DEPOIS da refatoracao

## Interpretacao dos argumentos

Leia os argumentos passados pelo usuario. Devem haver exatamente 2 caminhos de arquivos .md separados por espaco. O primeiro e o report "antes", o segundo e o report "depois".

Se os argumentos estiverem ausentes ou invalidos, pergunte ao usuario pelos caminhos.

## Execucao

### Passo 1 — Ler e extrair dados dos dois relatorios

Leia os dois arquivos markdown. De cada relatorio, extraia:

1. **Nome do projeto** — primeira linha apos `Project:`
2. **Stack** — linha apos `Stack:`
3. **Total de findings por severidade** — da linha `CRITICAL: X | HIGH: Y | MEDIUM: Z | LOW: W`
4. **Total de findings** — da linha `Total: N findings`
5. **Lista de findings** — cada bloco `### [SEVERITY] Nome do Anti-Pattern` com:
   - Severity (CRITICAL / HIGH / MEDIUM / LOW)
   - Nome do anti-pattern (texto apos a severidade)
   - File (linha apos `File:`)
   - Description (texto apos `Description:`)
   - Impact (texto apos `Impact:`)
   - Recommendation (texto apos `Recommendation:`)

### Passo 2 — Comparar findings

Compare os findings dos dois relatorios para determinar:

1. **Resolvidos** — findings presentes no "antes" mas NAO presentes no "depois". Use o nome do anti-pattern e a descricao para matching. Agrupe findings do mesmo tipo que foram resolvidos juntos (ex: "10x SQL Injection").
2. **Pendentes** — findings presentes no "depois" (podem ser novos ou continuacoes).
3. **Novos** — findings presentes no "depois" que nao existiam no "antes" (raros, mas possiveis).

Calcule:
- Total resolvidos = total antes - total depois (aproximado; ajuste se houver findings novos)
- Percentual de resolucao por severidade
- Percentual de resolucao geral

### Passo 3 — Agrupar por categoria de anti-pattern

Identifique as categorias de anti-patterns encontradas (ex: SQL Injection, Hardcoded Credentials, God Class, etc.) e para cada uma, conte quantos findings existiam antes e quantos restaram depois.

### Passo 4 — Gerar o dashboard HTML

Gere um arquivo HTML completo seguindo EXATAMENTE o template abaixo. Substitua os placeholders `<...>` pelos dados extraidos.

O dashboard deve ser salvo no mesmo diretorio do arquivo "depois", com o nome `dashboard-comparativo.html`.

**Template HTML obrigatoria (use este layout e estilo):**

```html-template
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Audit Dashboard — <NOME_PROJETO></title>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; background: #0f1117; color: #e1e4e8; }

  .header { background: linear-gradient(135deg, #1a1d27 0%, #252836 100%); padding: 40px; border-bottom: 1px solid #2d3140; }
  .header h1 { font-size: 28px; font-weight: 700; color: #fff; margin-bottom: 8px; }
  .header p { font-size: 14px; color: #8b8fa3; }

  .container { max-width: 1200px; margin: 0 auto; padding: 32px; }

  .metrics-row { display: grid; grid-template-columns: repeat(3, 1fr); gap: 16px; margin-bottom: 32px; }
  .metric-card { background: #1a1d27; border: 1px solid #2d3140; border-radius: 12px; padding: 24px; text-align: center; }
  .metric-value { font-size: 36px; font-weight: 700; color: #2ed573; }
  .metric-value.orange { color: #ff9f43; }
  .metric-value.red { color: #ff4757; }
  .metric-label { font-size: 12px; color: #8b8fa3; text-transform: uppercase; letter-spacing: 0.5px; margin-top: 4px; }

  .cards-row { display: grid; grid-template-columns: repeat(4, 1fr); gap: 16px; margin-bottom: 32px; }
  .card { background: #1a1d27; border: 1px solid #2d3140; border-radius: 12px; padding: 24px; position: relative; overflow: hidden; }
  .card::before { content: ''; position: absolute; top: 0; left: 0; right: 0; height: 3px; }
  .card.critical::before { background: #ff4757; }
  .card.high::before { background: #ff9f43; }
  .card.medium::before { background: #feca57; }
  .card.low::before { background: #54a0ff; }
  .card-label { font-size: 12px; text-transform: uppercase; letter-spacing: 1px; color: #8b8fa3; margin-bottom: 8px; }
  .card-values { display: flex; align-items: baseline; gap: 12px; }
  .card-before { font-size: 28px; font-weight: 700; color: #ff4757; }
  .card-arrow { font-size: 18px; color: #8b8fa3; }
  .card-after { font-size: 28px; font-weight: 700; }
  .card-after.improved { color: #2ed573; }
  .card-after.same { color: #ff9f43; }
  .card-diff { font-size: 12px; margin-top: 4px; }
  .card-diff.positive { color: #2ed573; }
  .card-diff.neutral { color: #8b8fa3; }

  .section-title { font-size: 18px; font-weight: 600; color: #fff; margin: 32px 0 16px 0; padding-bottom: 8px; border-bottom: 1px solid #2d3140; }

  .chart-row { display: grid; grid-template-columns: 1fr 1fr; gap: 24px; margin-bottom: 32px; }
  .chart-box { background: #1a1d27; border: 1px solid #2d3140; border-radius: 12px; padding: 24px; }
  .chart-box h3 { font-size: 14px; color: #8b8fa3; text-transform: uppercase; letter-spacing: 0.5px; margin-bottom: 20px; }
  .chart-box canvas { width: 100% !important; height: 220px !important; }

  .progress-section { margin-bottom: 32px; }
  .progress-bar-container { background: #1a1d27; border: 1px solid #2d3140; border-radius: 12px; padding: 24px; }
  .progress-item { margin-bottom: 16px; }
  .progress-item:last-child { margin-bottom: 0; }
  .progress-label { display: flex; justify-content: space-between; font-size: 13px; margin-bottom: 6px; }
  .progress-label span:first-child { color: #e1e4e8; }
  .progress-label span:last-child { color: #8b8fa3; }
  .progress-track { height: 8px; background: #2d3140; border-radius: 4px; overflow: hidden; }
  .progress-fill { height: 100%; border-radius: 4px; transition: width 1s ease; }
  .progress-fill.green { background: linear-gradient(90deg, #2ed573, #7bed9f); }
  .progress-fill.orange { background: linear-gradient(90deg, #ff9f43, #feca57); }
  .progress-fill.red { background: linear-gradient(90deg, #ff4757, #ff6b81); }

  .resolved-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 24px; margin-bottom: 32px; }
  .resolved-box, .remaining-box { background: #1a1d27; border: 1px solid #2d3140; border-radius: 12px; padding: 24px; }
  .resolved-box h3, .remaining-box h3 { font-size: 14px; text-transform: uppercase; letter-spacing: 0.5px; margin-bottom: 16px; display: flex; align-items: center; gap: 8px; }
  .resolved-box h3 { color: #2ed573; }
  .remaining-box h3 { color: #ff4757; }
  .resolved-box h3 .badge, .remaining-box h3 .badge { font-size: 11px; background: #2d3140; padding: 2px 8px; border-radius: 10px; color: #8b8fa3; }

  .finding-item { padding: 12px 0; border-bottom: 1px solid #2d3140; }
  .finding-item:last-child { border-bottom: none; }
  .finding-item .sev { display: inline-block; font-size: 10px; font-weight: 700; padding: 2px 8px; border-radius: 4px; text-transform: uppercase; margin-right: 8px; }
  .sev.critical { background: rgba(255,71,87,0.15); color: #ff4757; }
  .sev.high { background: rgba(255,159,67,0.15); color: #ff9f43; }
  .sev.medium { background: rgba(254,202,87,0.15); color: #feca57; }
  .sev.low { background: rgba(84,160,255,0.15); color: #54a0ff; }
  .finding-item .desc { font-size: 13px; color: #c0c4d0; margin-top: 4px; line-height: 1.5; }
  .finding-item .file { font-size: 12px; color: #8b8fa3; font-family: 'SF Mono', 'Fira Code', monospace; margin-top: 2px; }

  .footer { text-align: center; padding: 24px; font-size: 12px; color: #8b8fa3; border-top: 1px solid #2d3140; margin-top: 32px; }

  @media (max-width: 768px) {
    .cards-row, .chart-row, .resolved-grid, .metrics-row { grid-template-columns: 1fr; }
    .container { padding: 16px; }
  }
</style>
</head>
<body>
  <!-- HEADER -->
  <div class="header">
    <h1>Architecture Audit Dashboard</h1>
    <p><NOME_PROJETO> — Comparativo Antes vs Depois da Refatoracao</p>
  </div>

  <div class="container">

    <!-- METRICAS RESUMO (3 cards no topo) -->
    <div class="metrics-row">
      <div class="metric-card">
        <div class="metric-value"><PERCENTUAL_RESOLVIDO>%</div>
        <div class="metric-label">Problemas Resolvidos (<RESOLVIDOS> de <TOTAL_ANTES>)</div>
      </div>
      <!-- card 2: algo relevante como "Critical reduzidos" ou "Total findings" -->
      <!-- card 3: outro destaque relevante -->
    </div>

    <!-- CARDS POR SEVERIDADE -->
    <div class="cards-row">
      <!-- Para CADA severidade (CRITICAL, HIGH, MEDIUM, LOW), um card: -->
      <!--
      <div class="card critical">
        <div class="card-label">Critical</div>
        <div class="card-values">
          <span class="card-before"><ANTES></span>
          <span class="card-arrow">&rarr;</span>
          <span class="card-after improved"><DEPOIS></span>
        </div>
        <div class="card-diff positive">-<DIFERENCA> resolvidos (<PERCENT>% resolvido)</div>
      </div>
      -->
      <!-- Repetir para HIGH, MEDIUM, LOW -->
    </div>

    <!-- GRAFICOS DE BARRAS -->
    <div class="chart-row">
      <div class="chart-box">
        <h3>Distribuicao Antes</h3>
        <canvas id="chartBefore"></canvas>
      </div>
      <div class="chart-box">
        <h3>Distribuicao Depois</h3>
        <canvas id="chartAfter"></canvas>
      </div>
    </div>

    <!-- BARRAS DE PROGRESSO POR CATEGORIA -->
    <div class="progress-section">
      <div class="section-title">Resolucao por Categoria de Anti-Pattern</div>
      <div class="progress-bar-container">
        <!-- Para CADA categoria de anti-pattern identificada: -->
        <!--
        <div class="progress-item">
          <div class="progress-label"><span><CATEGORIA></span><span><ANTES> &rarr; <DEPOIS> (<PERCENT>% resolvido)</span></div>
          <div class="progress-track"><div class="progress-fill green|orange|red" style="width: <PERCENT>%"></div></div>
        </div>
        -->
        <!-- Cor: green se >60%, orange se 30-60%, red se <30% -->
      </div>
    </div>

    <!-- FINDINGS RESOLVIDOS vs PENDENTES -->
    <div class="resolved-grid">
      <div class="resolved-box">
        <h3><span style="color:#2ed573">&#10003;</span> Resolvidos <span class="badge"><COUNT> findings</span></h3>
        <!-- Para CADA finding resolvido: -->
        <!--
        <div class="finding-item">
          <span class="sev critical|high|medium|low">SEVERITY</span>
          <div class="desc">Descricao do que foi resolvido</div>
          <div class="file">arquivo_antes &rarr; arquivo_depois</div>
        </div>
        -->
      </div>

      <div class="remaining-box">
        <h3><span style="color:#ff4757">&#9888;</span> Pendentes <span class="badge"><COUNT> findings</span></h3>
        <!-- Para CADA finding pendente (do report "depois"): -->
        <!--
        <div class="finding-item">
          <span class="sev critical|high|medium|low">SEVERITY</span>
          <div class="desc">Descricao do problema</div>
          <div class="file">arquivo:linha</div>
        </div>
        -->
      </div>
    </div>

    <!-- DIAGNOSTICO FINAL -->
    <div class="section-title">Diagnostico Final</div>
    <div class="chart-box" style="margin-bottom: 0;">
      <p style="font-size: 14px; line-height: 1.8; color: #c0c4d0;">
        <!-- Paragrafo de 3-4 linhas resumindo o resultado da refatoracao. -->
        <!-- Destaque: percentual geral de resolucao, principais melhorias, areas que ainda precisam atencao. -->
        <!-- Use cores inline para destacar: <strong style="color: #2ed573;">para positivos</strong>, -->
        <!-- <strong style="color: #ff4757;">para criticos</strong>, <strong style="color: #ff9f43;">para alertas</strong> -->
      </p>
    </div>

  </div>

  <div class="footer">
    Gerado em <MES ANO> &mdash; Refactor Architect Audit
  </div>

  <!-- Chart.js para graficos -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.7/dist/chart.umd.min.js"></script>
  <script>
    const colors = { critical: '#ff4757', high: '#ff9f43', medium: '#feca57', low: '#54a0ff' };
    const chartOpts = {
      responsive: true,
      maintainAspectRatio: false,
      plugins: { legend: { display: false } },
      scales: {
        x: { ticks: { color: '#8b8fa3', font: { size: 11 } }, grid: { display: false } },
        y: { ticks: { color: '#8b8fa3', stepSize: 2, font: { size: 11 } }, grid: { color: '#2d3140' } }
      }
    };

    new Chart(document.getElementById('chartBefore'), {
      type: 'bar',
      data: {
        labels: ['Critical', 'High', 'Medium', 'Low'],
        datasets: [{ data: [<CRITICAL_ANTES>, <HIGH_ANTES>, <MEDIUM_ANTES>, <LOW_ANTES>], backgroundColor: [colors.critical, colors.high, colors.medium, colors.low], borderRadius: 6, barThickness: 40 }]
      },
      options: { ...chartOpts }
    });

    new Chart(document.getElementById('chartAfter'), {
      type: 'bar',
      data: {
        labels: ['Critical', 'High', 'Medium', 'Low'],
        datasets: [{ data: [<CRITICAL_DEPOIS>, <HIGH_DEPOIS>, <MEDIUM_DEPOIS>, <LOW_DEPOIS>], backgroundColor: [colors.critical, colors.high, colors.medium, colors.low], borderRadius: 6, barThickness: 40 }]
      },
      options: { ...chartOpts }
    });
  </script>
</body>
</html>
```

### Passo 5 — Salvar e abrir no browser

Salve o arquivo HTML gerado no mesmo diretorio do arquivo "depois", com o nome `dashboard-comparativo.html`.

Apos salvar, abra o dashboard no browser padrao do sistema. Detecte o OS e use o comando correto:

- **macOS:** `open <CAMINHO_ABSOLUTO_DO_DASHBOARD>`
- **Windows:** `start <CAMINHO_ABSOLUTO_DO_DASHBOARD>`
- **Linux:** `xdg-open <CAMINHO_ABSOLUTO_DO_DASHBOARD>`

Use o caminho absoluto do arquivo HTML gerado.

### Passo 6 — Confirmar

Imprima ao final:

```
================================
DASHBOARD GENERATED
================================
Project:    <nome do projeto>
Before:     <total antes> findings (<caminho arquivo antes>)
After:      <total depois> findings (<caminho arquivo depois>)
Resolved:   <total resolvidos> (<percentual>%)
Dashboard:  <caminho do arquivo HTML gerado>
================================
```

## REGRAS GERAIS

- Os dois argumentos sao obrigatorios. Se faltar algum, solicite ao usuario.
- O dashboard DEVE seguir o template HTML acima com o mesmo esquema de cores e layout.
- Todos os numeros devem ser consistentes — os totais nos cards devem bater com as listagens.
- Agrupe findings do mesmo tipo quando fizer sentido (ex: "3x SQL Injection" em vez de listrar 3 vezes).
- O diagnostico final deve ser um paragrafo conciso e concreto, sem generalidades.
- Use HTML entities (`&rarr;`, `&mdash;`, etc.) onde necessario.
- Nao inclua emojis no HTML.
- O arquivo deve ser abrivel diretamente no browser (HTML standalone com CDN para Chart.js).