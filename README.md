# 本地环境运行

```
conda create --name stanford_core_nlp python=3.7
conda activate stanford_core_nlp

# 安装指定包
pip install -r requirements.txt

# 安装NLP所需java
sudo apt-get install default-jdk
Windows电脑安装java离线包https://java.com/en/download/manual.jsp，随后配置高级系统设置，修改系统变量，新增JAVA_HOME,路径为C:\Program Files\Java\jre-1.8；编辑Path变量，在变量值的末尾添加 ;C:\Program Files\Java\jre-1.8\bin。重启Terminal，`java`测试能够调用java。

# 下载Stanford Core NLP
wget http://nlp.stanford.edu/software/stanford-corenlp-full-2018-10-05.zip
cd Measuring-Corporate-Culture-Using-Machine-Learning-master
unzip stanford-corenlp-full-2018-10-05.zip

# 修改配置，调用NLP
修改`global_options.py`，os.environ["CORENLP_HOME"] = "/home/user/stanford-corenlp-full-2018-10-05/"
Windows电脑修改"/"为"\\",os.environ["CORENLP_HOME"] = "C:\\user\\stanford-corenlp-full-2018-10-05"

# 降级protobuf
pip install protobuf==3.20.3

# 测试调用NLP
python -m culture.preprocess

# 在这里，碰到一个问题需要解决。如果文本过长，每一个row里的文本存在换行，可能会导致转换出来的documents.txt的行数不一致，导致错误。"AssertionError: Make sure the input file has the same number of rows as the input ID file"，运行下方`clean_documents.py`，按照json格式进行输出txt，确保行数一致，不受row内换行影响

import json
from openpyxl import load_workbook

def convert_xlsx_to_txt(input_xlsx_path, output_txt_path):
    # 加载 Excel 文件
    workbook = load_workbook(filename=input_xlsx_path, read_only=True)
    sheet = workbook.active

    # 获取初始行数
    initial_line_count = sheet.max_row
    print(f"Initial line count: {initial_line_count}")

    # 打开输出文件进行写入
    with open(output_txt_path, 'w', encoding='utf-8') as f:
        for row in sheet.iter_rows(values_only=True):
            # 将每行的第一个单元格值写入到文本文件
            item = row[0]
            if item is not None:
                # 将每个单元格内容作为 JSON 字符串写入文件，以确保换行符不会影响行数
                json_item = json.dumps(str(item))
                f.write(f"{json_item}\n")
            else:
                f.write("\n")

    # 验证输出文件的行数
    with open(output_txt_path, 'r', encoding='utf-8') as f:
        output_line_count = sum(1 for line in f)

    print(f"Output file line count: {output_line_count}")

    assert initial_line_count == output_line_count, (
        f"Line count mismatch: Excel file has {initial_line_count} lines, "
        f"but output file has {output_line_count} lines."
    )

    print(f"Conversion complete. {initial_line_count} lines written to {output_txt_path}")

if __name__ == "__main__":
    input_xlsx_path = 'full.xlsx'  # 替换为你的 Excel 文件路径
    output_txt_path = 'jason_output.txt'  # 替换为输出文本文件路径
    convert_xlsx_to_txt(input_xlsx_path, output_txt_path)


# 查看行数是否一致
wc -l documents.txt
wc -l document_ids.txt

# 运行分析
python parse_parallel.py
python clean_and_train.py
python create_dict.py
python score.py
python aggregate_firms.py

# 运行分析时，在Windows环境下，可能会报错`UnicodeEncodeError: 'gbk' codec can't encode character '\xa0' in position 14025: illegal multibyte sequence`
如报错，根据报错提示，去找到所在行数，添加encoding = "utf-8"即可。
此问题，主要涉及parse_parallel.py（52行，80行和83行）和parse.py（69行，97行和100行）


# Measuring Corporate Culture Using Machine Learning

## Introduction
The repository implements the method described in the paper 

Kai Li, Feng Mai, Rui Shen, Xinyan Yan, [__Measuring Corporate Culture Using Machine Learning__](https://academic.oup.com/rfs/advance-article-abstract/doi/10.1093/rfs/hhaa079/5869446?redirectedFrom=fulltext), _The Review of Financial Studies_, 2020; DOI:[10.1093/rfs/hhaa079](http://dx.doi.org/10.1093/rfs/hhaa079) 
[[Available at SSRN]](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3256608)

The code is tested on Ubuntu 18.04 and macOS Catalina, with limited testing on Windows 10.  

## Requirement
The code requres 
- `Python 3.6+`
- The required Python packages can be installed via `pip install -r requirements.txt`
- Download and uncompress [Stanford CoreNLP v3.9.2](http://nlp.stanford.edu/software/stanford-corenlp-full-2018-10-05.zip). Newer versions may work, but they are not tested. Either [set the environment variable to the location of the uncompressed folder](https://stanfordnlp.github.io/stanfordnlp/corenlp_client.html), or edit the following line in the `global_options.py` to the location of the uncompressed folder, for example: 
> os.environ["CORENLP_HOME"] = "/home/user/stanford-corenlp-full-2018-10-05/"   

- If you are using Windows, use "/" instead of "\\" to separate directories.  
- Make sure [requirements for CoreNLP](https://stanfordnlp.github.io/CoreNLP/) are met. For example, you need to have Java installed (if you are using Windows, install [Windows Offline (64-bit) version](https://java.com/en/download/manual.jsp)). To check if CoreNLP is set up correctly, use command line (terminal) to navigate to the project root folder and run `python -m culture.preprocess`. You should see parsed outputs from a single sentence printed after a moment:

    (['when[pos:WRB] I[pos:PRP] be[pos:VBD] a[pos:DT]....

## Data
We included some example data in the `data/input/` folder. The three files are
- `documents.txt`: Each line is a document (e.g., each earnings call). Each document needs to have line breaks remvoed. The file has no header row. 
- `document_ids.txt`: Each line is document ID (e.g., unique identifier for each earnings call). A document ID cannot have `_` or whitespaces. The file has no header row. 
- (Optional) `id2firms.csv`: A csv file with three columns (`document_id`:str, `firm_id`:str, `time`:int). The file has a header row. 


## Before running the code
You can config global options in the `global_options.py`. The most important options are perhaps:
- The RAM allocated for CoreNLP
- The number of CPU cores for CoreNLP parsing and model training
- The seed words
- The max number of words to include in each dimension. Note that after filtering and deduplication (each word can only be loaded under a single dimension), the number of words will be smaller. 


## Running the code
1. Use `python parse.py` to use Stanford CoreNLP to parse the raw documents. This step is relatvely slow so multiple CPU cores is recommended. The parsed files are output in the `data/processed/parsed/` folder:
    - `documents.txt`: Each line is a *sentence*. 
    - `document_sent_ids.txt`: Each line is a id in the format of `docID_sentenceID` (e.g. doc0_0, doc0_1, ..., doc1_0, doc1_1, doc1_2, ...). Each line in the file corresponds to `documents.txt`. 
    
    Note about performance: This step is time-consuming (~10 min for 100 calls). Using `python parse_parallel.py` can speed up the process considerably (~2 min with 8 cores for 100 calls) but it is not well-tested on all platforms. To not break things, the two implementations are separated. 

2. Use `python clean_and_train.py` to clean, remove stopwords, and named entities in parsed `documents.txt`. The program then learns corpus specific phrases using gensim and concatenate them. Finally, the program trains the `word2vec` model. 

    The options can be configured in the `global_options.py` file. The program outputs the following 3 output files:
    - `data/processed/unigram/documents_cleaned.txt`: Each line is a *sentence*. NERs are replaced by tags. Stopwords, 1-letter words, punctuation marks, and pure numeric tokens are removed. MWEs and compound words are concatenated. 
    - `data/processed/bigram/documents_cleaned.txt`: Each line is a *sentence*. 2-word phrases are concatenated.  
    - `data/processed/trigram/documents_cleaned.txt`: Each line is a *sentence*. 3-word phrases are concatenated. This is the final corpus for training the word2vec model and scoring. 

   The program also saves the following gensim models:
   - `models/phrases/bigram.mod`: phrase model for 2-word phrases
   - `models/phrases/trigram.mod`: phrase model for 3-word phrases
   - `models/w2v/w2v.mod`: word2vec model
   
3. Use `python create_dict.py` to create the expanded dictionary. The program outputs the following files:
    - `outputs/dict/expanded_dict.csv`: A csv file with the number of columns equal to the number of dimensions in the dictionary (five in the paper). The row headers are the dimension names. 
    
    (Optional): It is possible to manually remove or add items to the `expanded_dict.csv` before scoring the documents. 

4. Use `python score.py` to score the documents. Note that the output scores for the documents are not adjusted by the document length. The program outputs three sets of scores: 
    - `outputs/scores/scores_TF.csv`: using raw term counts or term frequency (TF),
    - `outputs/scores/scores_TFIDF.csv`: using TF-IDF weights, 
    - `outputs/scores/scores_WFIDF.csv`: TF-IDF with Log normalization (WFIDF). 

    (Optional): It is possible to use additional weights on the words (see `score.score_tf_idf()` for detail).  

5. (Optional): Use `python aggregate_firms.py` to aggregate the scores to the firm-time level. The final scores are adjusted by the document lengths. 
