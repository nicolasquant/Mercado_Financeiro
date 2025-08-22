## Modelo Quantitativo de Markowitz (Portfólios Otimizados)

Este repositório contém um script em Python que implementa, de forma prática, o modelo de seleção de portfólios de Harry Markowitz. A abordagem utiliza dados históricos do Yahoo Finance para:

- Estimar retornos esperados (anualizados)
- Estimar riscos (volatilidades, via matriz de covariâncias anualizada)
- Gerar milhares de portfólios aleatórios com restrição de soma de pesos igual a 1
- Visualizar o trade-off risco-retorno e a razão de Sharpe aproximada

### O que você vai encontrar
- Coleta de dados com `yfinance`
- Cálculo de retornos logarítmicos diários e anualização
- Geração de portfólios aleatórios e cálculo de risco x retorno
- Gráficos de séries de preço e do plano risco x retorno (colorido pela Sharpe aproximada)
- Funções utilitárias para estatísticas do portfólio
- Esqueleto de otimização (via `scipy.optimize`) para maximizar a Sharpe (com risco livre zero)


## Referencial teórico (resumo)

- Retorno esperado do portfólio: \( \mu_p = w^T \bar{r} \times 252 \), onde \( w \) são os pesos (soma 1) e \( \bar{r} \) é a média de retornos diários.
- Risco (volatilidade) do portfólio: \( \sigma_p = \sqrt{ w^T (\Sigma \times 252) \, w } \), onde \( \Sigma \) é a matriz de covariâncias diárias.
- Razão de Sharpe (sem taxa livre de risco): \( S = \mu_p / \sigma_p \). Caso queira incluir taxa livre de risco \( r_f \), use \( S = (\mu_p - r_f) / \sigma_p \).

Observação: adota-se 252 dias úteis por ano para anualização.


## Pré-requisitos

- Python 3.9+
- Bibliotecas: `numpy`, `pandas`, `matplotlib`, `yfinance`, `scipy`

Instale com:

```bash
pip install numpy pandas matplotlib yfinance scipy
```

Se preferir, crie e ative um ambiente virtual antes de instalar.


## Estrutura principal do script

- `stocks`: lista de tickers usados. Exemplo atual: `['AAPl', 'WMT', 'TSLA', 'GE', 'AMZN', 'DB']`
  - Atenção: há um erro de digitação comum em `AAPL` (Apple). Verifique se está exatamente como `AAPL`.
- `NUM_TRADING_DAYS`: constante para anualizar (padrão 252)
- `NUM_PORTFOLIOS`: quantos portfólios aleatórios serão gerados (padrão 10000)
- `start_date` e `end_date`: janela histórica para análise

### Funções

- `download_data()`: baixa preços de fechamento diários dos tickers com `yfinance` e retorna um `DataFrame` de preços.
- `show_data(data_frame)`: plota as séries de preços (fechamento) ao longo do tempo.
- `calculate_return(data)`: calcula retornos logarítmicos diários a partir dos preços.
- `show_statistics(returns)`: imprime estatísticas anualizadas por ativo (média e matriz de covariâncias).
- `show_mean_variance(returns, weights)`: exibe retorno e volatilidade anualizados de um portfólio dado um vetor de pesos.
- `generate_portfolios(returns)`: simula `NUM_PORTFOLIOS` carteiras aleatórias, cada uma com pesos que somam 1, e retorna pesos, retornos e riscos anualizados.
- `show_portfolios(returns, volatilities)`: gráfico de dispersão risco x retorno, colorido pela razão retorno/volatilidade (proxy de Sharpe sem \( r_f \)).
- `statistics(weights, inputs)`: para um vetor de pesos, retorna `[retorno, volatilidade, sharpe]` anualizados.
- `min_function_sharpe(weights, returns)`: função-alvo (negativa) para maximização da Sharpe via minimização.
- `optimize_portfolios(...)`: esqueleto de configuração de restrições e limites para otimização (ver seção Melhorias).


## Como rodar

1. Ajuste a lista de `stocks`, `start_date` e `end_date` conforme sua necessidade.
2. Execute o script Python.

Exemplo:

```bash
python seu_script.py
```

O script irá:
- Baixar os dados
- Exibir o gráfico de preços
- Calcular retornos logarítmicos e estatísticas anualizadas
- Gerar portfólios aleatórios, calcular risco/retorno e exibir o gráfico do plano risco x retorno


## Interpretação das saídas

- Gráfico de preços: evolução do preço de fechamento para cada ticker.
- Estatísticas: médias anualizadas por ativo e matriz de covariância anualizada.
- Gráfico risco x retorno: cada ponto é um portfólio; a cor representa aproximadamente a razão de Sharpe (maior é melhor, assumindo \( r_f = 0 \)).


## Melhorias e extensões

### 1) Corrigir ticker da Apple
- Se você pretende usar Apple, substitua `AAPl` por `AAPL` em `stocks`.

### 2) Incluir taxa livre de risco na Sharpe (opcional)
- Ajuste a função `statistics` para usar `portfolio_return - risk_free_rate` no numerador.

### 3) Otimização da Sharpe com `scipy.optimize`
O esqueleto já define limites e a restrição de soma de pesos igual a 1. Você pode finalizar a função de otimização conforme abaixo:

```python
def optimize_portfolios(returns):
    num_assets = len(stocks)
    constraints = ({'type': 'eq', 'fun': lambda w: np.sum(w) - 1})
    bounds = tuple((0.0, 1.0) for _ in range(num_assets))
    initial_guess = np.repeat(1.0 / num_assets, num_assets)

    result = optimization.minimize(
        fun=lambda w: -statistics(w, returns)[2],  # maximiza Sharpe (sem r_f)
        x0=initial_guess,
        method='SLSQP',
        bounds=bounds,
        constraints=constraints,
    )

    return result.x, result
```

Uso após calcular `retorno_diario_logaritmo`:

```python
best_weights, opt_result = optimize_portfolios(retorno_diario_logaritmo)
show_mean_variance(retorno_diario_logaritmo, best_weights)
print('Pesos ótimos (Sharpe máx):', best_weights)
```

### 4) Fronteira eficiente
- Você pode derivar a fronteira eficiente ao varrer retornos-alvo e resolver sucessivas otimizações de variância mínima com a restrição de retorno.

### 5) Restrições adicionais
- Limites por ativo, grupos, proibições de short, tracking error, custo de transação etc.


## Boas práticas de uso de dados

- Verifique a disponibilidade e a qualidade dos dados de `yfinance`. Falhas temporárias podem ocorrer.
- Confirme se os tickers estão corretos para o mercado/bolsa desejado (ex.: `AAPL` na NASDAQ; `PETR4.SA` na B3, etc.).
- Ajuste a janela temporal (`start_date`, `end_date`) conforme a estratégia de investimento.


## Resolução de problemas

- Erro 404/Conexão ao baixar dados: verifique conexão ou tente novamente mais tarde; considere cachear os dados.
- Série vazia/NaN: ticker incorreto ou período sem negociação suficiente.
- Gráficos não aparecem: rode em ambiente com backend gráfico habilitado ou salve as figuras com `plt.savefig(...)`.


## Aviso importante

Este código é para fins educacionais. Resultados históricos não garantem desempenho futuro. Antes de utilizar em produção, inclua validações mais robustas, tratamento de dados, métricas de risco adicionais e testes.
