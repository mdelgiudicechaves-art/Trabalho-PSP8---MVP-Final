# MVP — Espectroscopia de impedância para apoio à decisão em tecido mamário

Universidade de Brasília · Depto. de Engenharia de Produção · PSP8 (Prof. André Luiz Marques Serrano)

> Trabalho acadêmico sem validade clínica.

## 1. Objetivo e perguntas de negócio
Avaliar se medidas de impedância elétrica de tecido mamário excisado sustentam a triagem entre tipos de tecido (em particular carcinoma) e quais medidas vale a pena coletar.

- **P1.** Quais atributos de impedância mais diferenciam os seis tipos de tecido e quais são redundantes entre si?
- **P2.** Os seis tipos de tecido são separáveis por aprendizado supervisionado? Qual o desempenho (validação cruzada repetida e aninhada) e quais pares de tecidos são confundidos?
- **P3.** É possível triar **carcinoma × não carcinoma** com sensibilidade ≥ 90%? A que custo de especificidade, e qual seria o valor preditivo em prevalências realistas (1%, 5%, 20%)?
- **P4.** Os dados formam grupos naturais (aprendizado não supervisionado)? Eles coincidem com os tipos de tecido ou com o agrupamento em 3 grupos citado na literatura do dataset?
- **P5.** Quantas e quais variáveis bastam? Existe uma regra simples e interpretável (árvore rasa) com desempenho próximo ao do melhor modelo?

## 2. Dados
- **Fonte:** UCI Machine Learning Repository - Breast Tissue (Jossinet, 1996)
- **Licença:** CC BY 4.0 (UCI) (conferida em 11/11/2026)
- **Coleta:** 2026-11-11T17:31:57Z · arquivo `Breast Tissue Impedance Measurements - Data.csv` · 106 linhas × 10 colunas · 16361 bytes · formato CSV · sha256 `4d7d2601f6778db979e922b3ac115e67ff2a5a6b1dc19ddd279ef342b9b7a3df`

## 3. Pipeline
`CSV → Bronze (texto + metadados) → Silver (tipagem, regras R1–R6) → Gold (esquema estrela via SQL) → catálogo → análise P1–P5`
Plataforma desta execução: **colab** (DuckDB + Parquet). Diagrama: `catalogo/diagrama_estrela.png`; catálogo de dados: `catalogo/catalogo_dados.md`.

## 4. Como reproduzir
1. Abra `notebooks/MVP_Impedancia_Mamaria_Unificado.ipynb` no Google Colab.
2. Envie o CSV (upload no Colab).
3. Execute todas as células. Parâmetros na seção 2 do notebook.

## 5. Qualidade dos dados
| Dimensão | Teste | Afetados | Status |
|---|---|---|---|
| Completude | Nulos em qualquer atributo ou na classe | 0 | OK |
| Unicidade | Linhas repetidas (todos os atributos e a classe) | 1 | ALERTA |
| Unicidade | Mesmas medidas com classes diferentes (conflito de rótulo) | 0 | OK |
| Conformidade | Classe fora das 6 categorias válidas | 0 | OK |
| Conformidade | i0, da, max_ip, a_da ou p_curva <= 0 | 0 | OK |
| Conformidade | area < 0 | 0 | OK |
| Conformidade | pa500 fora de [0, π/2] | 0 | OK |
| Consistência | a_da diferente de area / da (tolerância relativa 1e-6) | 0 | OK |
| Consistência | p_curva < da (curva menor que a distância entre extremidades) | 0 | OK |
| Consistência | dr > da | 0 | OK |
| Consistência | dr < 0 (distância negativa) | 1 | ALERTA |
| Consistência | hfs < 0 (inclinação negativa) | 4 | INFO |
| Acurácia | Linhas com algum valor além de 3·IQR dentro da própria classe | 6 | ALERTA |
| Atualidade | Coluna de data/hora de medição | 0 | N/A |

## 6. Resultados
| Pergunta | Resultado principal |
|---|---|
| P1 | 9 atributos testados; mais discriminantes: P, I0, DA; 5 pares com |ρ| ≥ 0,90 |
| P2 | Regressão logística: macro-F1 0,68 ± 0,10; acurácia ao nível de 3 grupos 93% |
| P3 | AUC 0,978; sensibilidade 95% / especificidade 95% no limiar 0,37 |
| P4 | melhor k pelo silhouette = 2; ARI(k=6, tecidos) = 0,42 |
| P5 | 6 variável(is) dentro da tolerância; árvore de profundidade 3: macro-F1 0,48 |

Figuras:
- `evidencias/modelo_estrela.png`
- `evidencias/p1_boxplots_atributos.png`
- `evidencias/p1_efeito_correlacao.png`
- `evidencias/p2_modelos_confusao.png`
- `evidencias/p3_triagem_carcinoma.png`
- `evidencias/p4_agrupamento_pca.png`
- `evidencias/p5_parcimonia_arvore.png`
- `evidencias/qualidade_visao_geral.png`

### Discussão P1 — atributos que diferenciam os tecidos
- **Poder de discriminação:** 9 dos 9 atributos diferem entre as 6 classes (Kruskal–Wallis, p ajustado < 0,05). Os mais discriminantes são **P** (ε² = 0,82, muito forte), **I0** (ε² = 0,80, muito forte), **DA** (ε² = 0,78, muito forte). O menos discriminante é **HFS** (ε² = 0,17, relativamente forte).
- **Perfil por tecido:** **Adiposo** tem a maior mediana em 5 dos 9 atributos; **Carcinoma** tem a maior mediana em 2 dos 9 atributos; **Conjuntivo** tem a maior mediana em 2 dos 9 atributos. Isso mostra que a escala das medidas separa sobretudo tecidos de natureza muito diferente; para tecidos parecidos é preciso olhar a combinação de atributos (P2).
- **Redundância:** 5 pares de atributos têm |ρ| ≥ 0,90, por exemplo I0–P (ρ = 0,99), DA–DR (ρ = 0,97), Area–Max.IP (ρ = 0,92), Area–A.DA (ρ = 0,92), DA–Area (ρ = 0,91). Muitos atributos derivam do mesmo espectro (distâncias, área e comprimento da curva), portanto carregam **informação sobreposta**: modelos lineares sofrem com multicolinearidade e é possível reduzir o número de variáveis coletadas (testado em P5).
- **Ressalva:** com n = 105 e classes de 14 a 22 amostras, testes de significância têm pouco poder e p-valores pequenos não garantem generalização; os ε² são estimativas imprecisas.
### Discussão P2 — separabilidade dos seis tecidos
- **Resposta:** **parcialmente**: bem para alguns tecidos e mal para outros (ver abaixo). O melhor modelo foi **Regressão logística**, com **macro-F1 = 0,68 ± 0,10**, acurácia 70,4% e acurácia balanceada 69,2% (validação cruzada 5×10 com ajuste aninhado). O baseline (sempre a classe majoritária) fica em macro-F1 = 0,06; o ganho é de 0,62 pontos de F1.
- **Por classe:** maior sensibilidade em **Carcinoma** (90%) e menor em **Mastopatia** (36%). A sensibilidade para **carcinoma** é 90% no problema de 6 classes.
- **Pares confundidos:** Fibroadenoma × Mastopatia (28% das amostras do par trocadas); Mastopatia × Glandular (19% das amostras do par trocadas); Fibroadenoma × Glandular (15% das amostras do par trocadas).
- **Agrupamento em 3 grupos:** agrupando as predições em G1 (carcinoma), G2 (fibroadenoma+mastopatia+glandular) e G3 (conjuntivo+adiposo), a acurácia sobe de 70,4% (6 classes) para **92,8%**. Ou seja, boa parte do erro ocorre **dentro dos grupos**, o que sustenta o agrupamento da literatura como nível de decisão mais realista.
- **Limites:** n pequeno, média ± desvio entre dobras (que **não são independentes**) subestima a incerteza, e comparar 5 modelos na mesma validação favorece levemente o melhor (viés de seleção). A base não traz identificador de paciente: se houver várias amostras do mesmo paciente, a validação cruzada pode estar **otimista** (vazamento entre dobras).
### Discussão P3 — triagem de carcinoma
- **Discriminação:** o melhor modelo (Random Forest) atinge **AUC = 0,978** (IC95% bootstrap 0,95–1,00; média das repetições 0,973 ± 0,009), precisão média = 0,90 (referência de acaso = prevalência, 0,20). Para comparação, o melhor atributo isolado (**PA500**) tem AUC = 0,970: **um único atributo praticamente iguala o modelo multivariado** para esta pergunta (a escolha do melhor atributo foi feita sobre os próprios dados, portanto é levemente otimista).
- **Ponto de operação:** com limiar 0,37, a sensibilidade é **95,2%** (IC95% 83%–100%; 1 de 21 carcinomas não sinalizados) e a especificidade **95,2%** (IC95% 90%–99%; 4 falsos positivos). A meta de sensibilidade foi atingida. O limiar foi escolhido nas mesmas probabilidades em que é avaliado, portanto a sensibilidade/especificidade reais tendem a ser um pouco **menores**.
- **Onde estão os falsos positivos:** o tecido mais sinalizado indevidamente é **Mastopatia** (17% das amostras); os demais ficam entre 0% e 5%.
- **Valor preditivo depende da prevalência:** com essa sensibilidade/especificidade, o VPP seria **17%** a 1% de prevalência, **51%** a 5% de prevalência, **83%** a 20% de prevalência. Em um cenário de rastreamento (prevalência baixa) a maioria dos alertas seria falso; o teste serve como **apoio à decisão para amostras já suspeitas**, não como rastreamento populacional.
- **Limites:** 21 casos de carcinoma geram intervalos largos; amostras ex vivo; sem validação externa; custo relativo FN:FP não informado (a meta de 90% é uma premissa do aluno).
### Discussão P4 — grupos naturais
- **Número de grupos:** o silhouette do K-Means é máximo em **k = 2** (0,393 → *estrutura fraca*, escala de Kaufman & Rousseeuw); em k = 6 (o número de tecidos) cai para 0,317 (*estrutura fraca*).
- **Concordância com os tecidos:** com k = 6, ARI = **0,42** e NMI = 0,55 contra os 6 tecidos (ARI = 1 seria concordância perfeita; ≈ 0 seria acaso); contra os 3 grupos da literatura, o ARI do K-Means com k = 3 é **0,73**. Os grupos recuperam apenas parcialmente os tecidos: a estrutura natural é mais **grosseira** (poucos blocos) que a classificação em 6 tipos.
- **Estabilidade:** ARI entre sementes = 1,00 (k = 2) e 0,98 (k = 6), isto é, o resultado do K-Means é estável com `n_init = 20`.
- **Agrupamento em 3 blocos:** com k = 3 o K-Means concorda mais com os 3 grupos da literatura (ARI = 0,73) do que com os 6 tecidos (ARI = 0,42); isso é **evidência não supervisionada a favor do nível de 3 grupos** como resolução natural dos dados.
- **Leitura visual:** as duas primeiras componentes principais explicam 85% da variância; sobreposições na PCA entre tecidos próximos explicam por que o agrupamento não supervisionado não os separa, o que é coerente com os pares confundidos em P2.
- **Limite:** silhouette baixo não prova que não existam subtipos reais, apenas que a geometria dos atributos não os destaca com esse método e esse número de amostras.
### Discussão P5 — parcimônia e interpretabilidade
- **Variáveis mais importantes (permutação):** **I0** (0,062), **PA500** (0,046), **DA** (0,046). A importância é dividida entre variáveis correlacionadas (P1), por isso valores individuais subestimam o grupo.
- **Quantas bastam (6 classes):** com SVM (RBF), o macro-F1 máximo (0,69) ocorre com 8 variáveis; **6** já ficam a no máximo 0,02 do máximo (macro-F1 = 0,68; com todas as 9: 0,66). Variáveis mais frequentes no top-6 entre as dobras: I0 (100%), PA500 (100%), DA (100%), P (100%), Max.IP (93%), A.DA (80%). A redução é **pequena**: a curva sobe de forma gradual, ou seja, para separar 6 tecidos quase todas as medidas contribuem. Para coletar menos medidas, a seleção deve ser **reavaliada em dados independentes**.
- **Incerteza:** os desvios entre dobras da importância por permutação são comparáveis às médias (ver barras de erro); o ranking individual é **incerto**, sobretudo entre variáveis correlacionadas.
- **Regra interpretável:** uma árvore de profundidade 3 alcança macro-F1 = **0,48** (acurácia 58%) em validação cruzada, contra 0,68 do melhor modelo (Regressão logística). A melhor profundidade testada foi 5 (macro-F1 0,60). A diferença para o modelo completo é o preço da transparência: relevante; a regra simples serve para explicar, não para decidir sozinha.
- **Limite:** árvores são instáveis em amostras pequenas (outra amostra pode gerar outras regras); as regras impressas acima ilustram a lógica, não são limiares clínicos.
### Discussão geral — o que os resultados dizem sobre o problema
- **Técnica:** as medidas de impedância carregam informação: Regressão logística chega a macro-F1 = 0,68 em 6 classes e a 93% de acurácia no nível de 3 grupos. O agrupamento não supervisionado, porém, só enxerga estrutura grosseira (k = 2), então a separação fina entre tecidos parecidos depende de **rótulos e de modelos supervisionados**.
- **Triagem de carcinoma:** a relação sensibilidade × especificidade é **promissora**: com sensibilidade de 95% (IC95% 83%–100%) a especificidade é 95%; estimativas **otimistas** (limiar escolhido nos mesmos dados, 21 casos). Isso apoia o uso como **teste complementar** em amostras suspeitas, e não como substituto da histologia.
- **Operacional:** para separar os 6 tecidos são necessárias cerca de 6 das 9 medidas (ganho de parcimônia pequeno); já para **carcinoma × resto**, o melhor atributo isolado (PA500) tem AUC = 0,97, o que indica que a triagem pode exigir bem menos medidas que a classificação fina (a confirmar em dados independentes).
- **Viabilidade financeira (uso de recursos):** o pipeline completo (coleta, Bronze, Silver, Gold, validação cruzada e análises) levou **230 s** neste ambiente (colab), sobre 16,0 KB de dados brutos. É compatível com compute gratuito (Colab/Databricks Free Edition); o custo de operação é desprezível, e o custo real do projeto estaria na coleta das medidas.
- **Decisão recomendada:** seguir para uma fase 2 **somente** com (i) identificador de paciente para validação agrupada, (ii) conjunto de teste externo, (iii) calibração de probabilidades e definição do custo relativo FN:FP com especialistas clínicos.
### Autoavaliação

**1. Quais perguntas foram respondidas e quais não foram? Por quê?** Todas as cinco foram respondidas com evidência quantitativa, em graus diferentes de confiança:

- **P1 (atributos e redundância):** respondida (Kruskal–Wallis com ε²; 5 pares redundantes com |ρ| ≥ 0,90). Confiança moderada: n pequeno.

- **P2 (separabilidade):** respondida (macro-F1 = 0,68 com Regressão logística; 93% ao nível de 3 grupos). A distinção entre tecidos semelhantes ficou **apenas parcialmente resolvida**.

- **P3 (triagem de carcinoma):** respondida (sensibilidade 95%, especificidade 95%). A pergunta sobre **prevalências realistas** foi respondida por Bayes, mas sem dados reais de prevalência, só com cenários hipotéticos.

- **P4 (grupos naturais):** respondida (k = 2 pelo silhouette). Resultado **parcialmente negativo**: os grupos não reproduzem os 6 tecidos.

- **P5 (parcimônia):** respondida (≈ 6 de 9 variáveis para 6 classes; menos para carcinoma × resto). Seleção a ser confirmada em dados externos.

**2. Que limitações dos dados condicionaram os resultados?** (i) **Amostra pequena** (106 linhas; 14 a 22 por classe), que gera intervalos largos e favorece sobreajuste; (ii) **sem identificador de paciente**: se amostras do mesmo paciente caírem em dobras diferentes, a validação cruzada fica otimista; (iii) **medidas ex vivo** de uma única fonte/equipamento, sem validação externa; (iv) **prevalência artificial** (20% de carcinoma); (v) rótulos de categoria clínica e do agrupamento em 3 grupos vêm de documentação a confirmar; (vi) unidades da maioria dos atributos não são documentadas.

**3. Que decisões técnicas eu tomaria de outra forma?** (i) Obter o identificador de paciente e usar **validação cruzada agrupada**; (ii) reservar um **conjunto de teste final** antes de comparar modelos (evitar viés de seleção); (iii) definir **com especialistas o custo relativo FN:FP** antes de escolher o limiar, em vez de fixar 90% de sensibilidade; (iv) registrar experimentos com **MLflow** (disponível no Databricks); (v) usar **PySpark/Delta** também nas transformações da Silver (aqui feitas em pandas por serem 106 linhas, o que mantém o notebook portátil entre Colab e Databricks).

**4. Que extensões transformariam este MVP em solução de uso contínuo?** (i) Ingestão automática de novos lotes (Auto Loader/Jobs) com **data contracts** e testes de qualidade que falham o pipeline; (ii) **monitoramento de deriva** das medidas e de desempenho; (iii) calibração e **validação externa/prospectiva**; (iv) servir o modelo (API ou tabela de pontuação) com trilha de auditoria; (v) governança no Unity Catalog (permissões, linhagem automática), conformidade com a LGPD e **aprovação ética/regulatória** para qualquer uso clínico.

## 7. Ética, licenciamento e LGPD
Base pública, sem identificadores diretos nem de paciente; licença conforme acima; sem uso clínico.

## 8. Evidências de execução na nuvem
Inserir capturas de tela em `evidencias/` (ver checklist E1–E7 no notebook) e referenciá-las neste ponto do texto.
