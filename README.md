# Setup

```
conda create --name factscore python=3.10 --yes
conda activate factscore
pip install --editable .   # We need to use the code in this repo
python -m spacy download en_core_web_sm
python factscore/download_data.py   # No need for reconstructing instruct-llama-7b if we use openai
pip install gdown
gdown https://drive.google.com/uc?id=1mekls6OGOKLmt7gYtHs0WGf5oTamTNat -O .cache/factscore/enwiki-20230401.db  # See https://github.com/shmsw25/FActScore/issues/40#issue-2125580571
```

Put your OpenAI key in a file, assumed to be `openai_key.txt` here.


## Issues

- In the code, the atomic fact generator is InstructGPT and the factscore evaluator is (Retrieval+)ChatGPT.

- Originally, InstructGPT=GPT3=text-davinci-003, and ChatGPT=gpt-3.5-turbo.

- Due to deprecation, both InstructGPT and ChatGPT use gpt-3.5-turbo-instruct now. So the replication will not be exact.

- Some topics don't match exactly to titles in the DB due to ambiguity, which causes the code to fail (see [this issue](https://github.com/shmsw25/FActScore/issues/35)). An example is "Francisco Urroz". The code is modified to match titles with the same prefix if this happens (much slower), which gives "Francisco Urroz (rugby union)" and "Francisco Urroz (footballer)". The code then just uses all the paragraphs in these entries, assuming that the model shouldn't be penalized for the ambiguity of the query. This is an interesting example because GPT-4 is aware of the ambiguity at least (though it gets most facts wrong) while Alpaca-65B is oblivious, see `*-problem.jsonl`.


# Commands

The code will cache results on already completed prompts. By default these are `.cache/factscore/InstructGPT.pkl` for atomic fact generation and `.cache/factscore/ChatGPT.pkl` for factscore evaluation.

## Toy

Run atomic fact geneneration and factscore evaluation on 2 unlabeled biographies by ChatGPT.
```
python factscore/factscorer.py --input_path ChatGPT_unlabeled_head2.jsonl --model_name retrieval+ChatGPT --openai_key openai_key.txt --verbose
```

## Labeled

Evaluate the ChatGPT (i.e., the old gpt-3.5-turbo) responses, human-labeled with atomic facts. Only run the factscore evaluator (i.e., retrieval + gpt-3.5-turbo-instruct). Need to consume ~5.9 milion tokens corresponding to atomic facts with contexts in 157 ChatGPT biographies (I guess what's remaining after abstaining from the original 183 topics). Even though we don't need atomic fact generation, it can take a while for the OpenAI API to complete (2-3 hours). The cost is around $10.
```
python factscore/factscorer.py --input_path data/labeled/ChatGPT.jsonl --model_name retrieval+ChatGPT --openai_key openai_key.txt --use_atomic_facts --verbose
```

## Unlabeled

Evaluate the Alpaca-65B responses. Need to generate atomic facts as well as running the factscore evaluator (both by gpt-3.5-turbo-instruct). Alpaca-65B responds to 500 out of 500 topics.

For atomic fact generation, need to consume 2 million input tokens corresponding to in-context examples and the biography. Estimated cost $3, but this doesn't include output tokens so it'll be more like $6, I think.

For factscore evaluation, need to consume 8.1 million input tokens corresponding to atomic facts with contexts. Estimated cost $13. So the total cost is around $20.

The whole thing takes a few hours (OpenAI willing).


```
python factscore/factscorer.py --input_path data/unlabeled/Alpaca-65B.jsonl --model_name retrieval+ChatGPT --openai_key openai_key.txt --verbose
```
