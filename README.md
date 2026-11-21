# Projeto Aplicado II — Análise da dengue com sinais de alarme ou grave (SINAN 2025)

Guilherme Cunha Oliveira (RA 10754621) e Gabriel Lucca Simionato (RA 10751300) — Universidade Presbiteriana Mackenzie (EaD).

Objetivo (Mapeamento de Processos, Etapa 1): identificar perfis associados à classificação de dengue com sinais de
alarme ou dengue grave (`CLASSI_FIN` 11/12 vs 10), usando atributos dos casos (SINAN) e contexto municipal
(CNES leitos e IBGE Censo 2022), com a trilha textual dos Informes Semanais de Arboviroses como complemento.
A limpeza segue o **Plano de Limpeza das Bases** e as demais atividades o **Roteiro Complementar da Entrega 2** (ambos em `docs/`): brutos imutáveis, camada tratada separada, um notebook por fonte; depois EDA, preparação, treinamento, métricas e análise textual.

## Entrega final (Etapa 4)

| Item | Onde está |
|---|---|
| Relatório técnico cumulativo (Etapas 1 a 4) | `reports/Relatorio_Etapa4.pdf` 
| Storytelling (apresentação, 4 atos: problema, dados, solução, impacto) | `Storytelling_Dengue.pptx` na raiz do repositório  |
| Vídeo de apresentação (YouTube) | https://youtu.be/ZaMjrc9gDHQ |
| Código, dados tratados e modelo | este repositório (estrutura abaixo) |

**Resumo do projeto.** O modelo estima, para cada notificação de dengue, um escore de 0 a 1 para a chance de o caso evoluir para
sinais de alarme ou forma grave. O escore serve para **priorizar** a atenção da equipe de saúde; ele não diagnostica e não substitui a avaliação clínica.

**Resultado no conjunto de teste** (aberto uma única vez; Random Forest, limiar 0,463 escolhido na validação para recall alvo de 70%):

| Métrica | Valor |
|---|---|
| Recall (casos graves capturados) | 71,8% |
| Precisão | 7,9% |
| F1 | 0,142 |
| AUC-PR | 0,152 (regressão logística: 0,080) |
| AUC-ROC | 0,824 |
| Acurácia | 77,8% |
| Proporção de casos alertados | 23,3% |

Leitura prática: a cada 1.000 notificações, cerca de 25,5 evoluem para casos graves; o modelo emite cerca de 233 alertas e captura cerca de 18,3 desses casos.
Aproximadamente 92 de cada 100 alertas não evoluem para gravidade. Isso é o custo de manter a captura alta, e por isso o escore é um apoio à triagem, não uma decisão.

**Produto e modelo de negócio.** O painel de apoio à triagem (escore, faixa de alto e baixo risco) está **especificado no relatório (seções 13 e 14), mas não foi construído** nesta entrega.
O modelo de negócio (licença escalonada para secretarias de saúde e redes hospitalares) é uma **proposta**, sem valores definidos.

## Estrutura do repositório

```
db/
├── bases_brutas/        fontes originais, nunca sobrescritas (Git LFS)
│   ├── sinan/dados/  sinan/dicionario/
│   ├── cnes/
│   ├── ibge/  ibge/dicionario/       (mapeamento_indicadores.csv: água, esgoto, lixo — a aprovar)
│   └── informes/                     17 PDFs dos Informes Semanais do COE (nº 01–17, SE 1–4 a SE 1–23)
└── bases_tratadas/      saídas das limpezas + base analítica
    ├── sinan/     sinan_dengue_2025_tratado.csv (+ .parquet), _auditoria.csv, _dicionario.csv, colunas_decisao.csv, log
    ├── cnes/      cnes_municipio_2025_tratado.csv, cnes_estabelecimentos_2025_validado.csv, log
    ├── ibge/      ibge_municipio_2022_tratado.csv, log
    ├── informes/  informes_corpus_2025.csv, cabeçalhos removidos, seções detectadas, log
    └── analitica/ split_ids.csv (teste reservado), base_analitica_dev.parquet, métricas, resumo
notebooks/
├── 01_limpeza_sinan.ipynb          10 seções do plano: objetivo → resumo
├── 02_limpeza_cnes.ipynb
├── 03_limpeza_ibge.ipynb
├── 04_limpeza_informes.ipynb
├── 05_integracao_eda_modelo.ipynb  P03–P05 e P07: reserva do teste, EDA (7 itens do roteiro), preparação, baseline, métricas
├── 06_analise_informes.ipynb       P06: termos, localidades, sorotipos, tendências, TF-IDF, amostra para revisão manual
└── 07_modelo_final.ipynb           P07–P08: regressão logística vs Random Forest, escolha do limiar, avaliação única no teste, escore de risco
scripts/          um .py por notebook (gerados das células de código) + executar_tudo.sh
reports/          relatório técnico de cada etapa e figuras
diario/           diário de andamento
docs/             mapeamento de processos, plano de limpeza, materiais de apoio
```

## Como reproduzir

```bash
pip install -r requirements.txt
git lfs pull                      # traz os arquivos brutos reais
bash scripts/executar_tudo.sh     # SINAN (~2 min, 1,5 GB RAM) -> CNES -> IBGE -> Informes -> integração/modelo (~1,5 min) -> análise textual
# ou execute os notebooks na ordem 01 -> 02 -> 03 -> 04 -> 05 -> 06 -> 07
```

Padrão de processo em todas as fontes: **entrada** (base bruta preservada) → **processamento** (regras documentadas na seção 6 de cada
notebook) → **validação** (seção 8) → **saída** (camada tratada, com log). Semente fixa `SEED = 42`; o conjunto de teste (20%,
estratificado) é reservado em `db/bases_tratadas/analitica/split_ids.csv` antes de qualquer exploração e foi aberto uma única vez (notebook 07).

## Dados e método em resumo

- **Fontes:** SINAN/Dengue 2025 (1.643.215 notificações; 1.481.802 casos elegíveis, 37.792 positivos, 2,55%), CNES Leitos 2025, IBGE Censo 2022 e 17 Informes Semanais do COE.
- **Alvo:** `CLASSI_FIN` 10 → 0 (dengue); 11 e 12 → 1 (dengue com sinais de alarme ou grave).
- **Divisão:** teste 20% estratificado (296.361 casos; 7.558 graves); treino 948.352 e validação 237.089.
- **Pré-processamento:** imputação (moda com indicador para sintomas; mediana para variáveis contínuas), padronização Z-score ajustada só no treino, one-hot, colunas com risco de vazamento excluídas.
- **Modelos:** regressão logística com pesos de classe (método inicial do Mapeamento) e Random Forest (100 árvores, profundidade 12, mínimo de 50 por folha), adotado como ajuste por ter AUC-PR maior na validação.
- **Limiar:** 0,463, o maior valor que mantém recall ≥ 70% na validação.
- **Informes:** usados como contexto nacional (1.478.752 casos prováveis até a SE 23; 1.165 óbitos confirmados) e na trilha textual (TF-IDF); o modelo preditivo usa apenas a trilha tabular.

## Bibliotecas

pandas e NumPy (bases tabulares) · Matplotlib (gráficos) · scikit-learn (pré-processamento, separação, regressão logística, Random Forest, métricas, TF-IDF) · pdfplumber (informes) · openpyxl (dicionário do IBGE) · pyarrow (parquet) · biblioteca padrão (pathlib, hashlib, json, re).
Versões em `requirements.txt`.

## Exigências da Etapa 2 e onde estão atendidas

| Exigência | Evidência |
|---|---|
| Bibliotecas Python e GitHub | `requirements.txt`, este README, seção 2 do relatório |
| Análise exploratória | notebook 05 (Etapas 4–12) e `db/bases_tratadas/analitica/` |
| Tratamento / preparação e treinamento | notebooks 01–04 (limpeza) e 05 (Etapa 13: preparação e ajuste inicial) |
| Bases teóricas do método | seção 5 do relatório |
| Cálculo da acurácia | notebook 05 (Etapa 14) e seção 6 do relatório |

## Exigências da Etapa 3 e onde estão atendidas

| Exigência | Evidência |
|---|---|
| (a) Aplicação do método analítico | notebook 07 (Seções 2 a 4) e seção 8 do relatório |
| (b) Medidas de acurácia | notebook 07 (Seções 4 e 5) e seção 9 do relatório |
| (c) Resultados preliminares, produto e modelo de negócios | notebook 07 (Seções 5 e 6) e seção 10 do relatório |
| (d) Esboço do storytelling | notebook 07 (Seção 7) e seção 11 do relatório |

## Exigências da Etapa 4 e onde estão atendidas

| Exigência | Evidência |
|---|---|
| Relatório técnico cumulativo | `reports/Relatorio_Etapa4.docx` (seções 1 a 12 = Etapa 3; seções 13 a 17 = Etapa 4) |
| Produto final | seção 13 do relatório (painel de apoio à triagem, especificado) |
| Modelo de negócio | seção 14 do relatório (Business Model Canvas e projeções para 1.000 e 10.000 notificações) |
| Storytelling final | `Storytelling_Dengue.pptx` e seção 15 do relatório |
| Conclusão e pendências | seção 16 do relatório |
| Links (GitHub e vídeo) | seção 17 do relatório e tabela "Entrega final" deste README |

## Estado das etapas

| Processo do mapa | Situação |
|---|---|
| P01 Delimitar a proposta | concluído (Etapa 1) |
| P02 Ler e compreender as fontes | concluído — notebooks 01 a 04 |
| P03 Casos elegíveis e reserva do teste | concluído — 01 (alvo) e 05 (split) |
| P04 Explorar os dados de desenvolvimento | concluído — 05 |
| P05 Preparar e integrar | concluído para SINAN + CNES + IBGE básico; saneamento aguarda `mapeamento_indicadores.csv` aprovado |
| P06 Trilha textual (informes) | concluído — corpus, menções, TF-IDF e indicadores citados dos 17 informes; revisão manual da amostra pendente |
| P07 Ajustar e comparar na validação | concluído — baseline (05) e comparação Regressão Logística vs Random Forest com escolha do limiar (07) |
| P08 Avaliar no teste e comunicar | concluído — teste aberto uma vez (07), escore de risco, modelo de negócio, storytelling final (`Storytelling_Dengue.pptx`) e vídeo |

## Limitações e pendências

- O painel de apoio à triagem está especificado, mas não foi implementado.
- Os indicadores de saneamento do IBGE não entraram no modelo (aguardam aprovação do mapeamento).
- A revisão manual da amostra da trilha textual está pendente.
- O modelo de negócio é uma proposta; os valores de licença não foram definidos.
- O escore prioriza casos, não diagnostica, e deve ser usado junto com a avaliação clínica. Em uso real, o tratamento de dados de saúde deve seguir a LGPD (Lei 13.709/2018).
