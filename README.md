<h1 align="center">MultiWOZ and SpokenWOZ Evaluation</h1>

<h3 align="left">Standardized evaluation for MultiWOZ dialogue systems</h3>
<h3 align="left">Dialogue state tracking evaluation for Speech-aware MultiWoz and SpokenWoz </h3>

_______

### About this repository:

This is a fork of [Tomiinek/MultiWOZ_Evaluation](https://github.com/Tomiinek/MultiWOZ_Evaluation) with extensions for dialogue state tracking evaluation for spoken dialogue datasets.

Provides standardized evaluation on the [MultiWOZ benchmark](https://github.com/budzianowski/multiwoz) including:
- **Response generation**: BLEU, Inform & Success rates, lexical richness
- **Dialog state tracking**: Joint goal accuracy, slot F1/precision/recall, error rate analysis
- **Speech-aware datasets**: Support for [Speech-Aware MultiWOZ](https://aclanthology.org/2023.dstc-1.25/) and [SpokenWOZ](https://dl.acm.org/doi/10.5555/3666122.3667821)
- **Fuzzy matching**: Handles deviations in slot values via fuzzy matching: `fuzzywuzzy`

# :rocket: Usage

#### Install the repository:

``` sh
git clone https://github.com/BUTSpeechFIT/MultiWOZ_Evaluation.git
cd MultiWOZ_Evaluation
uv venv
uv pip install -r requirements.txt
```

**Use it directly from your code.** Instantiate an evaluator and then call the `evaluate` method with dictionary of your predictions with a specific format ([described later](#input-format)). Set `bleu` to evaluate the BLEU score, `success` to get the Success & Inform rate, and use `richness` for getting lexical richness metrics such as the number of unique unigrams, trigrams, token entropy, bigram conditional entropy, corpus MSTTR-50, and average turn length. Pseudo-code:

``` python
from mwzeval.metrics import Evaluator
...

e = Evaluator(bleu=True, success=False, richness=False)
my_predictions = {}
for item in data:
    my_predictions[item.dialog_id] = model.predict(item)
    ...

results = e.evaluate(my_predictions)
print(f"Epoch {epoch} BLEU: {results}")
```

Evaluate your predictions from the input file:

``` sh
python evaluate.py [--bleu] [--success] [--richness] [--dst] --input INPUT.json [--output OUTPUT.json] [--golden GOLDEN.json]
```
Set the options `--bleu`, `--success`, `--richness`, and `--dst` as you wish. For DST evaluation, use `--golden` to specify the ground truth reference file.

## :speech_balloon: Spoken DST Evaluation Pipeline

For automated evaluation on speech-aware datasets with fuzzy matching:

``` sh
bash eval_wrapper.sh PREDICTIONS.json DATASET OUTPUT_DIR [NUM_JOBS]
```

Where:
- `PREDICTIONS.json` - Your model predictions
- `DATASET` - One of: `sa_dev`, `sa_test_v` (verbatim), `sa_test_p` (paraphrased), `sp_dev`, `sp_test`
    - `sa`: speech-aware multiwoz
    - `sp`: spoken-woz
- `OUTPUT_DIR` - Directory for output results and log
- `NUM_JOBS` - Optional, number of parallel jobs for fuzzy matching (default: 4)

This performs both exact match and fuzzy match evaluation based on ontology and database of entity names for speech-aware dialogue systems.

See [mwzeval/data/database/](mwzeval/data/database/)


#### Input format:

``` python
{
    "xxx0000" : [
        {
            # response can be empty if doing only evaluation of DST
            "response": "Your generated delexicalized response.",
            "state": {
                "restaurant" : {
                    "food" : "eatable"
                }, ...
            },
            "active_domains": ["restaurant"]
        }, ...
    ], ...
}
```
The input to the evaluator should be a dictionary (or a `.json` file) with keys matching dialogue ids in the `xxx0000` format (e.g. `sng0073` instead of `SNG0073.json`), and values containing a list of turns. Each turn is a dictionary with keys:

- **`response`** – Your generated delexicalized response. You can use either the slot names with domain names, e.g. `restaurant_food`, or the domain adaptive delexicalization scheme, e.g. `food`.
- **`state`** – **Optional**, the predicted dialog state. If not present (for example in the case of policy optimization models), the ground truth dialog state from MultiWOZ 2.2 is used during the Inform & Success computation. Slot names and values are normalized prior the usage.
- **`active_domains`** – **Optional**, list of active domains for the corresponding turn. If not present, the active domains are estimated from changes in the dialog state during the Inform & Success rate computation. If your model predicts the domain for each turn, place them here. If you use domains in slot names, run the following command to extract the active domains from slot names automatically:

    ``` sh
    python add_slot_domains.py [-h] -i INPUT.json -o OUTPUT.json
    ```

See the [`predictions`](predictions) folder with examples.


#### Output format:

``` python
{
    "bleu" : {'damd': … , 'uniconv': … , 'hdsa': … , 'lava': … , 'augpt': … , 'mwz22': … },
    "success" : {
        "inform"  : {'attraction': … , 'hotel': … , 'restaurant': … , 'taxi': … , 'total': … , 'train': … },
        "success" : {'attraction': … , 'hotel': … , 'restaurant': … , 'taxi': … , 'total': … , 'train': … },
    },
    "richness" : {
        'entropy': … , 'cond_entropy': … , 'avg_lengths': … , 'msttr': … ,
        'num_unigrams': … , 'num_bigrams': … , 'num_trigrams': …
    }
}
```
The evaluation script outputs a dictionary with keys `bleu`, `success`, `richness`, and `dst` corresponding to BLEU, Inform & Success rates, lexical richness metrics, and dialogue state tracking metrics, respectively. Their values can be `None` if not evaluated, otherwise:

- **BLEU** results contain multiple scores corresponding to different delexicalization styles and refernces. Currently included references are DAMD, HDSA, AuGPT, LAVA, UniConv, and **MultiWOZ 2.2** whitch we consider to be the canonical one that should be reported in the future.
- **Inform & Succes** rates are reported for each domain (i.e. attraction, restaurant, hotel, taxi, and train in case of the test set) separately and in total.
- **Lexical richness** contains the number of distinct uni-, bi-, and tri-grams, average number of tokens in generated responses, token entropy, conditional bigram entropy, and MSTTR-50 calculated on concatenated responses.
- **DST** results include joint goal accuracy, slot error rate, slot F1/precision/recall, with detailed insertion/substitution/deletion error analysis. Fuzzy matching (95% threshold) is used for partial credit on slot values.


#### Dialog State Tracking Evaluation

You can use this code for comprehensive evaluation of dialogue state tracking (DST) on MultiWOZ 2.2, speech-aware MultiWoZ and SpokenWoz. Set `dst=True` during initialization of the `Evaluator` to get:
- Joint goal accuracy (exact state match)
- Slot-level F1, precision, and recall
- Slot error rate with insertion/substitution/deletion breakdown
- Optional fuzzy matching of slot values with the ones from database.

Note that the resulting numbers may differ from the original MultiWOZ DST evaluation due to our use of slot name/value normalization and careful fuzzy matching (95% threshold by default).

# :clap: Contributing

- **If you would like to add your results**, modify the particular table in the [original reposiotry](https://github.com/budzianowski/multiwoz) via a pull request, add the file with predictions into the `predictions` folder in this repository, and create another pull request here.
- **If you need to update the [slot name mapping](https://github.com/Tomiinek/MultiWOZ_Evaluation/blob/29512dec6df009e6b579a4aa8d26f8c1c6e85e35/normalization.py#L36-L55)** because of your different delexicalization style, feel free to make the changes, and create a pull request.
- **If you would like to improve [normalization of slot values](https://github.com/Tomiinek/MultiWOZ_Evaluation/blob/29512dec6df009e6b579a4aa8d26f8c1c6e85e35/normalization.py#L63-L254)**, add your new rules, and create a pull request.

# :book: Citation

If you use this evaluation toolkit, please cite the paper:

```bibtex
@inproceedings{nekvinda-dusek-2021-shades,
    title = "Shades of {BLEU}, Flavours of Success: The Case of {M}ulti{WOZ}",
    author = "Nekvinda, Tom{\'a}{\v{s}} and Du{\v{s}}ek, Ond{\v{r}}ej",
    booktitle = "Proceedings of the 1st Workshop on Natural Language Generation, Evaluation, and Metrics (GEM 2021)",
    month = aug,
    year = "2021",
    address = "Online",
    publisher = "Association for Computational Linguistics",
    url = "https://aclanthology.org/2021.gem-1.4",
    doi = "10.18653/v1/2021.gem-1.4",
    pages = "34--46"
}
```

If you use the speech-aware dialogue extensions (DST evaluation, fuzzy matching, etc.), please also cite:

```bibtex
@inproceedings{sedlacek25_interspeech,
    title = "Approaching Dialogue State Tracking via Aligning Speech Encoders and {LLM}s",
    author = "\v{S}imon Sedl\'{a}\v{c}ek and Bolaji Yusuf and J\'{a}n \v{S}vec and Pradyoth Hegde and Santosh Kesiraju and Old\v{r}ich Plchot and Jan \v{C}ernock\'{y}",
    booktitle = "Proc. INTERSPEECH 2025",
    year = "2025",
    url = "https://www.isca-archive.org/interspeech_2025/sedlacek25_interspeech.html"
}

@misc{vendrame2025jointspeechtexttraining,
    title = {Joint Speech and Text Training for {LLM}-Based End-to-End Spoken Dialogue State Tracking},
    author = {Katia Vendrame and Bolaji Yusuf and Santosh Kesiraju and \v{S}imon Sedl\'{a}\v{c}ek and Old\{r}ich Plchot and Jan \v{C}ernock\'{y}},
    year = {2025},
    eprint = {2511.22503},
    archivePrefix = {arXiv},
    primaryClass = {cs.CL},
    url = {https://arxiv.org/abs/2511.22503}
}
```

