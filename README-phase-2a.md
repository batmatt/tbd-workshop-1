IMPORTANT ❗ ❗ ❗ Please remember to destroy all the resources after each work session. You can recreate infrastructure by creating new PR and merging it to master.

![img.png](doc/figures/destroy.png)

0. The goal of this phase is to create infrastructure, perform benchmarking/scalability tests of sample three-tier lakehouse solution and analyze the results using:
* [TPC-DI benchmark](https://www.tpc.org/tpcdi/)
* [dbt - data transformation tool](https://www.getdbt.com/)
* [GCP Composer - managed Apache Airflow](https://cloud.google.com/composer?hl=pl)
* [GCP Dataproc - managed Apache Spark](https://spark.apache.org/)
* [GCP Vertex AI Workbench - managed JupyterLab](https://cloud.google.com/vertex-ai-notebooks?hl=pl)

Worth to read:
* https://docs.getdbt.com/docs/introduction
* https://airflow.apache.org/docs/apache-airflow/stable/index.html
* https://spark.apache.org/docs/latest/api/python/index.html
* https://medium.com/snowflake/loading-the-tpc-di-benchmark-dataset-into-snowflake-96011e2c26cf
* https://www.databricks.com/blog/2023/04/14/how-we-performed-etl-one-billion-records-under-1-delta-live-tables.html

2. Authors:


   Zespół 11
   - Mateusz Winnicki
   - Bartosz Sweklej
   - Magdalena Lutyńska

   [link to forked repo](https://github.com/batmatt/tbd-workshop-1)
   
4. Sync your repo with https://github.com/bdg-tbd/tbd-workshop-1.

   ![obraz](https://github.com/batmatt/tbd-workshop-1/assets/62250240/f5e2895c-8625-4bfa-ad38-2bfd6bc63460)

6. Provision your infrastructure.

    a) setup Vertex AI Workbench `pyspark` kernel as described in point [8](https://github.com/bdg-tbd/tbd-workshop-1/tree/v1.0.32#project-setup) 

    b) upload [tpc-di-setup.ipynb](https://github.com/bdg-tbd/tbd-workshop-1/blob/v1.0.36/notebooks/tpc-di-setup.ipynb) to 
the running instance of your Vertex AI Workbench

7. In `tpc-di-setup.ipynb` modify cell under section ***Clone tbd-tpc-di repo***:

   a)first, fork https://github.com/mwiewior/tbd-tpc-di.git to your github organization.

   b)create new branch (e.g. 'notebook') in your fork of tbd-tpc-di and modify profiles.yaml by commenting following lines:
   ```  
        #"spark.driver.port": "30000"
        #"spark.blockManager.port": "30001"
        #"spark.driver.host": "10.11.0.5"  #FIXME: Result of the command (kubectl get nodes -o json |  jq -r '.items[0].status.addresses[0].address')
        #"spark.driver.bindAddress": "0.0.0.0"
   ```
   This lines are required to run dbt on airflow but have to be commented while running dbt in notebook.

   c)update git clone command to point to ***your fork***.

 


8. Access Vertex AI Workbench and run cell by cell notebook `tpc-di-setup.ipynb`.

    a) in the first cell of the notebook replace: `%env DATA_BUCKET=tbd-2023z-9910-data` with your data bucket.


   b) in the cell:
         ```%%bash
         mkdir -p git && cd git
         git clone https://github.com/mwiewior/tbd-tpc-di.git
         cd tbd-tpc-di
         git pull
         ```
      replace repo with your fork. Next checkout to 'notebook' branch.
   
    c) after running first cells your fork of `tbd-tpc-di` repository will be cloned into Vertex AI  enviroment (see git folder).

    d) take a look on `git/tbd-tpc-di/profiles.yaml`. This file includes Spark parameters that can be changed if you need to increase the number of executors and
  ```
   server_side_parameters:
       "spark.driver.memory": "2g"
       "spark.executor.memory": "4g"
       "spark.executor.instances": "2"
       "spark.hadoop.hive.metastore.warehouse.dir": "hdfs:///user/hive/warehouse/"
  ```


7. Explore files created by generator and describe them, including format, content, total size.

   Aby prześledzić co zawierają wygenerowane dane zdecydowanie łatwiej jest zacząć od przeanalizowania [plików konfiguracyjnych generatora](https://github.com/batmatt/tbd-tpc-di/tree/main/tools/pdgf/config). 
   Plik `tpc-di-schema.xml` definiuje strukturę danych, która ma być generowana. Określa różne tabelki danych, ich pola oraz typy danych, które mają być w nich zawarte. Określa także zależności między różnymi tabelami oraz relacje, jakie mają zachodzić pomiędzy danymi oraz definiuje szczegółowe wartości pól, takich jak formaty numerów, łańcuchy znaków czy wartości logiczne.
   Wygenerowane pliki zostały podzielone na 3 foldery: Batch1, Batch2, Batch3. W każdym z nich znajdują się pliki .csv, z danymi np. `TradeType_audit.csv` o typach transakcji, `TaxRate_audit.csv` o stawkach podatkowych, `Account_audit.csv` o kontach. Następnie przy użyciu [skryptu](https://github.com/batmatt/tbd-tpc-di/blob/main/tpcdi.py) zawartość plików jest odczytywana i konwertowana na format kompatybilny z Hive tworząc w kubełku GCS Data Lakehouse. 

   ![img.png](doc/figures/generated_data_1.png)
   ![img.png](doc/figures/generated_data_2.png)

   Największe rozmiarowo są pliki `DailyMarket.txt` (3GB) zawiera dane o cenach akcji na przestrzeni czasu, `Trade.txt` (1.2GB), `TradeHistory.txt` (1GB) `CashTransactions.txt` (1GB) oraz `WatchHistory.txt` (1.3GB), zawierają one dane o transakcjach, których jak można się spodziewać podczas operowania rynku jest znacznie więcej niż innych, dotyczących np. typów transakcji, stawek podatkowych czy kont biorących udział w transakcjach. 

   W skrócie opis zawartości plików:
   - TaxRate.txt - id, nazwa i wartość stawki podatkowej
   - HR.csv - dane pracownika oraz id managera
   - WatchHistory.txt - dane o obserwowanych transakcjach przez klienta
   - Trade.txt - id, status, stempel czasowy, czy gotówkowa, ilość zakupionych akcji, wylicytowana cena i cena transakcji, stawka opodatkowania, id konta klienta
   - TradeHistory.txt - id transakcji, stempel czasowy, status transakcji
   - StatusType.txt - zbiór statusów transakcji
   - TradeType.txt - zbiór typów transakcji
   - HoldingHistory.txt - id, id transakcji po której nastąpiła zmiana holdingu, ilość akcji przed i po
   - CashTransactions.txt - id konta klienta, stempel czasowy, suma, nazwa

8. Analyze tpcdi.py. What happened in the loading stage?

   Skrypt automatyzuje ładowanie danych finansowych TPC-DI do hurtowni. Identyfikuje pliki, przenosi je do obszaru tymczasowego w Google Cloud Storage, tworzy DataFrame'y PySpark i wykonuje podstawowe przekształcenia.

   1. Definicja schematu: Każdy typ pliku ma zdefiniowany schemat za pomocą StructType i StructField dla strukturyzowanego ładowania danych.
   2. Przesyłanie plików: Pliki są przesyłane do GCS za pomocą klienta Google Cloud Storage, zapewniając ich dostępność dla Sparka.
   3. Tworzenie DataFrame: Przesłane pliki są wczytywane do DataFrames Sparka z odpowiednimi opcjami ustawionymi dla delimitera i schematu.
   4. Przetwarzanie DataFrame: Specyficzne transformacje i selekcje są wykonywane na DataFrames w celu wyodrębnienia i sformatowania wymaganych danych.
   5. Tworzenie tabel: Przetworzone DataFrames są pokazywane (jeśli show jest prawdziwe) lub zapisywane jako tabele Hive za pomocą funkcji save_df.

9. Using SparkSQL answer: how many table were created in each layer?

   ![img.png](doc/figures/table_in_layer_counter.png)

10. Add some 3 more [dbt tests](https://docs.getdbt.com/docs/build/tests) and explain what you are testing. ***Add new tests to your repository.***

   [commit with added tests](https://github.com/batmatt/tbd-tpc-di/commit/3f3ee88bbedd0b3ddadbda6f5ac43090281ea6fe)

11. In main.tf update
   ```
   dbt_git_repo            = "https://github.com/mwiewior/tbd-tpc-di.git"
   dbt_git_repo_branch     = "main"
   ```
   so dbt_git_repo points to your fork of tbd-tpc-di. 

   [commit with repo change](https://github.com/batmatt/tbd-workshop-1/commit/325f0d6d2eaea88129373497326b42c41b52f4c5) 
   [commit with branch change](https://github.com/batmatt/tbd-workshop-1/commit/282e2feb4c85f2359e070f72d052999f185d72da)

12. Redeploy infrastructure and check if the DAG finished with no errors:

