# Arquitectura de GenPath Explorer

GenPath Explorer é unha aplicación de escritorio que internamente funciona a través da interacción entre 7 subsistemas diferenciados. Un host de escritorio permite que esta se execute como un programa común, facilitando o acceso ao usuario non técnico, mentres que o frontend real corre nun servidor local como aplicativo web.

No backend, interactúan os módulos de interface de servizos (respondables de expoñer os endpoints HTTP e mediar coas solicitudes do frontend) co dominio e lóxica de negocio, onde se implementan as principais funcionalidades e cálculos realizados pola aplicación, e os módulos de memoria, persistencia de datos e servizos externos.

<img width="1564" height="1562" alt="mermaid-piia2" src="https://github.com/user-attachments/assets/229b53f8-e459-48d9-8674-58247a69874d" />
## Subsistemas da arquitectura

1. **Host de Escritorio**
   - Encárgase do ciclo de vida completo da aplicación e de envolvela como un programa nativo de escritorio. A súa función principal é abstraer ao usuario final de calquera configuración técnica: coordina o arranque ordenado dos servizos internos, garante que o servidor local estea dispoñible antes de mostrar a interface e asegura un peche limpo dos procesos en segundo plano cando se pecha a xanela.
   - Exemplos no código: *main.py* executa *main()*, onde instancia a base de datos, chama a *seed_sample_folders()* para cargar mostras iniciais, levanta o servidor ASGI local (uvicorn.Server en 127.0.0.1:8000) nun fío secundario e abre a xanela nativa con *webview.create_window()* e *webview.start()*.

2. **Frontend**
   - É a interface gráfica de usuario (SPA en React) encargada da visualización interactiva de táboas, gráficos, filtros e resultados. Execútase dentro do motor web integrado de pywebview (aínda que tamén pode abrirse nun navegador estándar en local) e mantén un desacoplamento total respecto aos datos: non accede directamente a ficheiros nin á base de datos, senón que toda a súa comunicación co backend realízase vía HTTP.
   - Exemplos no código: *frontend/index.html* carga os *assets* compilados de React, mentres que no código fonte (*web/src/api/client.js*) centralízanse as peticións JSON e multipart/form-data contra a API local.

3. **Interface de Servizos (API REST)**
   - Actúa como porta de entrada controlada ao backend. Expón os *endpoints* HTTP mediante FastAPI, aplica contratos estritos de validación de entradas e saídas con Pydantic e traduce os erros de dominio a códigos HTTP comprensibles para o cliente (400, 404, 422, 502). Deste xeito, a lóxica bioinformática queda illada dos detalles do protocolo web.
   - Exemplos no código:
     - *data.py* e *data_models.py*: xestionan datasets, subida e edición de ficheiros (*import_files*, *update_file*), preferencias, presets de filtros (*save_filter_preset*) e a descarga da plantilla CSV (*download_template*).
     - *analysis.py* e *analysis_models.py*: expoñen as vistas analíticas (*get_analysis*, *get_pair*, *get_pathway*, *get_gene_appearances*, *get_recurring_genes*).
     - *explanation.py* e *explanation_models.py*: xestionan provedores, chaves API (*set_api_key*), axustes (*update_settings*) e o *endpoint* *get_explanation*.
     - *loading.py*: centraliza en *load_for_request()* a validación do dataset e a carga filtrada dos tipos celulares para *analysis.py* e *explanation.py*.

4. **Dominio e Lóxica de Negocio**
   - Contén o núcleo funcional e bioinformático da aplicación. Aquí resólvense a lectura tolerante a erros de táboas heteroxéneas, as regras estatísticas de filtrado, os cálculos comparativos entre tipos celulares, a xeración de feitos estruturados para os modelos de linguaxe e o descubrimento de provedores LLM.
   - Exemplos no código:
     - *file_processing.py*: *read_file()* le Excel/CSV/TSV/JSON coma texto puro; *process_file()* detecta columnas (*detect_columns*), tipo celular (*detect_cell_type*) e colección (*infer_collection*), validando os campos obrigatorios.
     - *samples.py*: *seed_sample_folders()* importa automaticamente subcartafoles de datos de mostra que aínda non fosen procesados.
     - *filters.py* e *src/analysis.py*: *passes_filters()* e *is_significant()* aplican limiares de significancia, |NES| e dirección; o módulo de análise calcula patróns (*find_patterns*), similitudes (*compare_all_pairs*) e xenes recorrentes (*find_recurring_genes*).
     - *facts.py*: *build_facts()* transforma os resultados de análise nun JSON determinista con redondeos controlados (*round_nes*, *round_significance*, *round_percent*).
     - *llm_providers.py*: *get_models()*, *get_gemini_models()* e *get_groq_models()* filtran modelos de texto válidos e xestionan a obtención de chaves.

5. **Estado Compartido en Memoria (AppContext)**
   - Funciona como un contedor de dependencias en memoria inxectado en app.state.context e compartido por todas as peticións HTTP. A súa existencia evita reinstanciar recursos pesados ou repetir lecturas de disco e consultas de rede en cada chamada, mellorando o rendemento xeral da aplicación.
   - Exemplos no código: AppContext (creado en *main.py*) mantén tres referencias clave: a instancia de Database, GeneAnnotations (que carga *hgnc_genes.json* de forma preguiceira a primeira vez que se chama a *get(symbol)* en *gene_annotations.py*) e models_cache (dicionario en memoria para evitar consultas repetidas ás APIs de Gemini e Groq).

6. **Persistencia Local**
   - Abarca todo o almacenamento que reside na máquina do usuario baixo un enfoque *local-first*. Isto inclúe tanto os datos xerados durante o uso (datasets, filas orixinais e preferencias) coma os recursos estáticos empaquetados coa aplicación e a custodia segura de credenciais sen gardalas en texto plano.
   - Exemplos no código:
     - Database (SQLite): garda datasets, ficheiros coas súas filas en bruto (raw_rows), rutas procesadas, control de versións (data_version), cartafoles de mostra xa importados e preferencias.
     - *paths.py*: resolve rutas de datos de usuario (APP_DATA_DIR), recursos empaquetados (BUNDLE_DIR, FRONTEND_DIR, GENE_ANNOTATIONS_PATH) e cartafoles de mostras (*get_sample_dirs()*).
     - keyring: custodia as chaves API no chaveiro seguro do sistema operativo mediante *keyring.set_password()* e *keyring.get_password()*.

7. **Servizos Externos**
   - Representa a fronteira de rede fóra do equipo do usuario. Mentres que a maior parte da aplicación opera completamente *offline*, este subsistema xestiona a comunicación HTTPS con provedores externos de intelixencia artificial, illando os seus posibles modos de fallo (erros de rede, *timeouts* ou chaves rexeitadas).
   - Exemplos no código: As APIs HTTPS de Google Gemini (generativelanguage.googleapis.com) e Groq (api.groq.com), consultadas actualmente por *fetch_json()* en *llm_providers.py* para listar e validar os modelos dispoñibles.

---

## Fluxos principais de información

1. **Fluxo de Arranque e *Seeding***
   - *main.py* instancia Database e chama a *seed_sample_folders()* (*samples.py*) coas rutas devoltas por *get_sample_dirs()* (*paths.py*).
   - Para cada subcartafol non rexistrado na BD, *samples.py* le cada ficheiro con *read_file()* e normalízao con *process_file()* (*file_processing.py*), gardando as rutas e as filas en bruto en SQLite.
   - A continuación, *main.py* instancia GeneAnnotations e AppContext, arranca uvicorn en 127.0.0.1:8000 e abre pywebview cargando *frontend/index.html*.

2. **Fluxo de Inxesta e Remapeo de Ficheiros**
   - Subida: O Frontend envía ficheiros vía POST /import ou POST /dataset/{id}/files a *data.py* (*import_files_into_dataset*). Cada ficheiro pasa por *read_file()* e *process_file()* (*file_processing.py*), gárdase en SQLite con *database.save_file()* (incluíndo raw_rows) e devólvese o estado de cada ficheiro xunto co novo data_version.
   - Remapeo sen resubir: Se o usuario corrixe columnas ou a fonte de significancia (PUT /file/{file_id}), *update_file()* en *data.py* recupera raw_rows de SQLite, reconstrúe o DataFrame con *raw_rows_to_table()*, reexecuta *process_file()* co novo mapeo e actualiza a base de datos incrementando data_version.

3. **Fluxo de Análise Bioinformática**
   - O Frontend envía un POST con filtros (por exemplo, AnalysisFilters ou PairRequest) a *analysis.py*.
   - O *endpoint* delega en *load_for_request()* (*loading.py*), que verifica o dataset en SQLite e carga os tipos celulares filtrados segundo *filters.py*.
   - *analysis.py* executa as funcións de cálculo do módulo de análise (como *find_patterns*, *compare_pair*, *common_genes* ou *find_recurring_genes*). No *endpoint* *get_gene_appearances()*, consulta adicionalmente *context.gene_annotations.get()* (*gene_annotations.py*) para engadir metadatos HGNC.
   - Finalmente, devolve a resposta validada por Pydantic (*analysis_models.py*) incluíndo data_version para sincronización coa interface.

4. **Fluxo de Configuración de Provedores e Explicación**
   - Chaves e modelos: Cando o usuario garda unha chave (PUT /explanation/providers/{provider}/api-key), *set_api_key()* en *explanation.py* almacénaa en keyring e limpa a entrada dese provedor en context.models_cache. Ao listar modelos (GET .../models) ou validar axustes (PUT /explanation/settings), *get_models()* en *llm_providers.py* consulta primeiro models_cache; se non está en caché, le a chave de keyring, consulta a API externa de Gemini ou Groq vía HTTPS, filtra os modelos de texto e garda o resultado en caché.
   - Xeración de explicación: Ante un POST /explanation/{dataset_id} con ComparisonFilters, *get_explanation()* en *explanation.py* chama a *load_for_request()* e pasa os datos a *build_facts()* (*facts.py*) para construír o JSON real de feitos. Na versión actual, combina eses facts reais co *placeholder* (DUMMY_EXPLANATION e *random_verification()*) e devolve ExplanationResponse.
   -
   - ```

