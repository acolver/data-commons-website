# Tools and Data for Stat Var Embeddings Index

This directory contains the data sheets (containing StatVar DCID and
descriptions) and script used to construct the Stat Var Embeddings Index that
is loaded into the NL Server in Website.

There are multiple sizes of indexes: small, medium.

As of May 2023, the small index has ~1.3K variables and medium index has 5.3K
variables.

## Making a change to the embeddings.

1. For any carefully curated SVs/topics, make edits to the [curated input CSVs](data/curated_input/)<a name="curated-input"></a>.
   - The columns in this csv are:

      - `dcid`: the StatVar DCID.
      - `Name`: the name of the StatVar
      - `Description`: the description(s) of the StatVar. Multiple descriptions are acceptable. If multiple description strings are provided, they must be semi-colon delimited. This column can be expected to be used as part of any automated alternative generation processed, e.g. using an LLM.
      - `Override_Alternatives`: if this column has any value in a row, all other columns are ignored. Multiple strings are acceptable for this column. If multiple strings are provided, they must be semi-colon delimited.
      - `Curated_Alternatives`: a semi-colon delimited list of strings which can serve as alternative ways of referring to the StatVar.

   -  To easily edit the curated csv in Google sheets, you can go use the sheets command-line tools [here](../../sheets/). 
      - E.g., To copy the curated input for the main embeddings to a google sheet, go to the sheets command line tools folder and run:
      ```bash
      ./run.sh -m csv2sheet -l ../nl/embeddings/data/curated_input/main/sheets_svs.csv [-s <sheets_url>] [-w <worksheet_name>]
      ```
      - E.g., To copy the contents of the google sheet back as the curated input for the main embeddings, go to the sheets command line tools folder and run:
      ```bash
      ./run.sh -m sheet2csv -l ../nl/embeddings/data/curated_input/main/sheets_svs.csv -s <sheets_url> -w <worksheet_name>
      ```

2. Add any auto-generated input CSVs to `data/autogen_input/<index-size>/`.
   This is typically generated by code [here](prep/).

   Note: This is what distinguishes between the differently sized (small, medium) embeddings.

3. Ensure any updated alternatives, i.e. PaLM alternatives, Other alternatives, are available as csv files: [`palm_alternatives.csv`](data/alternatives/main/palm_alternatives.csv), [`other_alternatives.csv`](data/alternatives/main/other_alternatives.csv). The columns in these CSV files are: `dcid`, `Alternatives`. These files are expected to be updated using (currently) separate processes. <a name="alternatives"></a>

4. Run the command below which will generate a new embeddings csv in
   `gs://datcom-nl-models`. Note down the embeddings file version printed at the end of the run. As of 2023 Q3, there are 3 embeddings used in PROD and the following are the commands to generate each of them:

   To generate the `small` embeddings:
   ```bash
   ./run.sh -b small
   ```

   To generate the `medium_ft` embeddings:
   ```bash
   ./run.sh -f medium
   ```

   To generate the `sdg_ft` embeddings:
   ```bash
   ./run.sh -c sdg data/curated_input/sdg/sheets_svs.csv data/alternatives/sdg/*.csv
   ```

   You can also create custom embeddings (using the finetuned model in PROD):
   ```bash
   ./run.sh -c <embeddings_size> <curated_input_csv_path> <alternatives_filepattern>
   ```

   Notes:
   - curated_input_csv_path is the filepath of the CSV with the curated input to use. The format of the CSV should follow the description of [point 1](#curated-input).
   - alternatives_filepattern is the filepattern of the CSV files with the alternatives to use. The format of the CSVs should follow the description of [point 3](#alternatives).
   - Example: `./run.sh -c test data/curated_input/test/sheets_svs.csv data/alternatives/test/*.csv`

5. Validate the CSV diffs, update [`embeddings.yaml`](../../../deploy/nl/embeddings.yaml) with the generated embeddings version and test out locally.

6. Generate an SV embeddings differ report by following the process under the [`sv_index_differ`](../svindex_differ/README.md) folder (one level up). Look at the diffs and evaluate whether they make sense.

7. Remove embedding caches by running `rm -rf ~/.datacommons/cache.*` and then update goldens by running `./run_test.sh -g` from the repo root.

8. If everything looks good, send out a PR with the `embeddings.yaml`, the `differ_report.html` file (as a linked attachement), CSV changes, and updated goldens.

## Production Config Files

### [`embeddings.yaml`](../../../deploy/nl/embeddings.yaml)

Lists the embeddings CSV files (generated using the steps above).

The keys are index names (specified as `idx=` param value), and the values are file names (with the assumption that the files are stored in gs://datcom-nl-models/).

These files, generated from a fine-tuned model (as of Q2 2023), have the following structure:  `<version>.<fine-tuned-model-version>.<base-model-name>.csv`  (e.g., `datcom-nl-models/embeddings_sdg_2023_09_12_16_38_04.ft_final_v20230717230459.all-MiniLM-L6-v2.csv`).

### [`models.yaml`](../../../deploy/nl/models.yaml)

A mostly legacy file that lists the fine-tuned model name.

## One time setup

To allow the `gspread` library access to the google sheets above, you will need [credentials downloaded to your computer](https://docs.gspread.org/en/latest/oauth2.html#for-end-users-using-oauth-client-id).

As of Feb 2023, you can download the gspread-python-app credentials [found here](https://pantheon.corp.google.com/apis/credentials/oauthclient/878764285063-2tqmvvstv8k8cdl7ougccd7ptpnat8d5.apps.googleusercontent.com?project=datcom-204919) to `~/.config/gspread/credentials.json`.
