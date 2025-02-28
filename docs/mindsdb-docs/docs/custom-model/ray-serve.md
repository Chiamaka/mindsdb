# MindsDB and Ray Serve

Ray Serve is a simple high-throughput model serving library that can wrap around your ML model.

## Simple Example of Logistic Regression

In this example, we train an external scikit-learn model to use for making predictions.

### Creating the Ray Serve Model

Let's look at an actual model wrapped by a class that complies with the requirements.

```python
import ray
from fastapi import Request, FastAPI
from ray import serve
import time
import pandas as pd
import json
from sklearn.linear_model import LogisticRegression


app = FastAPI()
ray.init()
serve.start(detached=True)


async def parse_req(request: Request):
    data = await request.json()
    target = data.get('target', None)
    di = json.loads(data['df'])
    df = pd.DataFrame(di)
    return df, target


@serve.deployment(route_prefix="/my_model")
@serve.ingress(app)
class MyModel:
    @app.post("/train")
    async def train(self, request: Request):
        df, target = await parse_req(request)
        feature_cols = list(set(list(df.columns)) - set([target]))
        self.feature_cols = feature_cols
        X = df.loc[:, self.feature_cols]
        Y = list(df[target])
        self.model = LogisticRegression()
        self.model.fit(X, Y)
        return {'status': 'ok'}

    @app.post("/predict")
    async def predict(self, request: Request):
        df, _ = await parse_req(request)
        X = df.loc[:, self.feature_cols]
        predictions = self.model.predict(X)
        pred_dict = {'prediction': [float(x) for x in predictions]}
        return pred_dict


MyModel.deploy()

while True:
    time.sleep(1)
```

It is important to have the `/train` and `/predict` endpoints.

The `/train` endpoint accepts two parameters to be sent via POST:

- `df` is a serialized dictionary that can be converted into a pandas dataframe.
- `target` is the name of the target column to be predicted.

It returns a JSON object containing the `status` key and the `ok` value.

The `/predict` endpoint requires one parameter to be sent via POST:

- `df` is a serialized dictionary that can be converted into a pandas dataframe.

It returns a dictionary containing the `prediction` key. It stores the predictions. Additional keys can be returned for confidence and confidence intervals.

### Bringing the Ray Serve Model to MindsDB

Once you start the RayServe-wrapped model, you can create and train it in MindsDB.

```sql
CREATE PREDICTOR mindsdb.byom_ray_serve
FROM mydb (
    SELECT number_of_rooms, initial_price, rental_price 
    FROM test_data.home_rentals
) 
PREDICT number_of_rooms
USING
    url.train = 'http://127.0.0.1:8000/my_model/train',
    url.predict = 'http://127.0.0.1:8000/my_model/predict',
    dtype_dict={"number_of_rooms": "categorical", "initial_price": "integer", "rental_price": "integer"},
    format='ray_server';
```

Now, you can fetch predictions using the standard MindsDB syntax. Follow the guide on the [`SELECT`](/sql/api/select/) statement to learn more.

You can directly pass input data in the `WHERE` clause to get a single prediction.

```sql
SELECT *
FROM byom_ray_serve
WHERE initial_price=3000
AND rental_price=3000;
```

Or you can `JOIN` the model wth a data table to get bulk predictions.

```sql
SELECT tb.number_of_rooms, t.rental_price
FROM mydb.test_data.home_rentals AS t
JOIN mindsdb.byom_ray_serve AS tb
WHERE t.rental_price > 5300;
```

!!! TIP "Limit for POST Requests"
    Please note that if your model is behind a reverse proxy like nginx, you might have to increase the maximum limit for POST requests in order to receive the training data. MindsDB can send as much as you'd like - it has been stress-tested with over a billion rows.

## Example of Keras NLP Model

Here, we consider a [natural language processing (NLP) task](https://www.kaggle.com/c/nlp-getting-started) where we want to train a neural network using [Keras](https://keras.io) to detect if a tweet is related to a natural disaster, such as fires, earthquakes, etc. Please download [this dataset](https://www.kaggle.com/c/nlp-getting-started) to follow the example.

### Creating the Ray Serve Model

We create a Ray Serve service that wraps around the [Kaggle NLP Model](https://www.kaggle.com/shahules/basic-eda-cleaning-and-glove) that can be trained and used for making predictions.

```python
import re
import time
import json
import string
import requests
from collections import Counter, defaultdict
​
import ray
from ray import serve
​
import gensim
import numpy as np
import pandas as pd
from tqdm import tqdm
from nltk.util import ngrams
from nltk.corpus import stopwords
from nltk.tokenize import word_tokenize
from fastapi import Request, FastAPI
from sklearn.model_selection import train_test_split
from sklearn.feature_extraction.text import CountVectorizer
​
from tensorflow.keras.preprocessing.text import Tokenizer
from tensorflow.keras.preprocessing.sequence import pad_sequences
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Embedding, LSTM, Dense, SpatialDropout1D
from tensorflow.keras.initializers import Constant
from tensorflow.keras.optimizers import Adam
​
app = FastAPI()
stop = set(stopwords.words('english'))
​
​
async def parse_req(request: Request):
    data = await request.json()
    target = data.get('target', None)
    di = json.loads(data['df'])
    df = pd.DataFrame(di)
    return df, target
​
​
@serve.deployment(route_prefix="/nlp_kaggle_model")
@serve.ingress(app)
class Model:
    MAX_LEN = 100
    GLOVE_DIM = 50
    EPOCHS = 10
​
    def __init__(self):
        self.model = None
​
    @app.post("/train")
    async def train(self, request: Request):
        df, target = await parse_req(request)
​
        target_arr = df.pop(target).values
        df = self.preprocess_df(df)
        train_corpus = self.create_corpus(df)
​
        self.embedding_dict = {}
        with open('./glove.6B.50d.txt', 'r') as f:
            for line in f:
                values = line.split()
                word = values[0]
                vectors = np.asarray(values[1:], 'float32')
                self.embedding_dict[word] = vectors
        f.close()
​
        self.tokenizer_obj = Tokenizer()
        self.tokenizer_obj.fit_on_texts(train_corpus)
​
        sequences = self.tokenizer_obj.texts_to_sequences(train_corpus)
        tweet_pad = pad_sequences(sequences, maxlen=self.__class__.MAX_LEN, truncating='post', padding='post')
        df = tweet_pad[:df.shape[0]]
​
        word_index = self.tokenizer_obj.word_index
        num_words = len(word_index) + 1
        embedding_matrix = np.zeros((num_words, self.__class__.GLOVE_DIM))
​
        for word, i in tqdm(word_index.items()):
            if i > num_words:
                continue
​
            emb_vec = self.embedding_dict.get(word)
            if emb_vec is not None:
                embedding_matrix[i] = emb_vec
​
        self.model = Sequential()
        embedding = Embedding(num_words,
                              self.__class__.GLOVE_DIM,
                              embeddings_initializer=Constant(embedding_matrix),
                              input_length=self.__class__.MAX_LEN,
                              trainable=False)
        self.model.add(embedding)
        self.model.add(SpatialDropout1D(0.2))
        self.model.add(LSTM(64, dropout=0.2, recurrent_dropout=0.2))
        self.model.add(Dense(1, activation='sigmoid'))
​
        optimizer = Adam(learning_rate=1e-5)
        self.model.compile(loss='binary_crossentropy', optimizer=optimizer, metrics=['accuracy'])
​
        X_train, X_test, y_train, y_test = train_test_split(df, target_arr, test_size=0.15)
        self.model.fit(X_train, y_train, batch_size=4, epochs=self.__class__.EPOCHS, validation_data=(X_test, y_test), verbose=2)
​
        return {'status': 'ok'}
​
    @app.post("/predict")
    async def predict(self, request: Request):
        df, _ = await parse_req(request)
​
        df = self.preprocess_df(df)
        test_corpus = self.create_corpus(df)
​
        sequences = self.tokenizer_obj.texts_to_sequences(test_corpus)
        tweet_pad = pad_sequences(sequences, maxlen=self.__class__.MAX_LEN, truncating='post', padding='post')
        df = tweet_pad[:df.shape[0]]
​
        y_pre = self.model.predict(df)
        y_pre = np.round(y_pre).astype(int).flatten().tolist()
        sub = pd.DataFrame({'target': y_pre})
​
        pred_dict = {'prediction': [float(x) for x in sub['target'].values]}
        return pred_dict
​
    def preprocess_df(self, df):
        df = df[['text']]
        df['text'] = df['text'].apply(lambda x: self.remove_URL(x))
        df['text'] = df['text'].apply(lambda x: self.remove_html(x))
        df['text'] = df['text'].apply(lambda x: self.remove_emoji(x))
        df['text'] = df['text'].apply(lambda x: self.remove_punct(x))
        return df
​
    def remove_URL(self, text):
        url = re.compile(r'https?://\S+|www\.\S+')
        return url.sub(r'', text)
​
    def remove_html(self, text):
        html = re.compile(r'<.*?>')
        return html.sub(r'', text)
​
    def remove_punct(self, text):
        table = str.maketrans('', '', string.punctuation)
        return text.translate(table)
​
    def remove_emoji(self, text):
        emoji_pattern = re.compile("["
                                   u"\U0001F600-\U0001F64F"  # emoticons
                                   u"\U0001F300-\U0001F5FF"  # symbols & pictographs
                                   u"\U0001F680-\U0001F6FF"  # transport & map symbols
                                   u"\U0001F1E0-\U0001F1FF"  # flags (iOS)
                                   u"\U00002702-\U000027B0"
                                   u"\U000024C2-\U0001F251"
                                   "]+", flags=re.UNICODE)
        return emoji_pattern.sub(r'', text)
​
    def create_corpus(self, df):
        corpus = []
        for tweet in tqdm(df['text']):
            words = [word.lower() for word in word_tokenize(tweet) if ((word.isalpha() == 1) & (word not in stop))]
            corpus.append(words)
        return corpus
​
​
if __name__ == '__main__':
​
    ray.init()
    serve.start(detached=True)
​
    Model.deploy()
​
    while True:
        time.sleep(1)
```

Now, we need access to the training data. For that, we create a table called `nlp_kaggle_train` to load the [dataset](https://www.kaggle.com/c/nlp-getting-started) that the original model uses. The `nlp_kaggle_train` table contains the following columns:

```sql
id INT,
keyword VARCHAR(255),
location VARCHAR(255),
text VARCHAR(5000),
target INT
```

Please note that the specifics of the schema/table and how to ingest the CSV data vary depending on your database.

### Bringing the Ray Serve Model to MindsDB

Now, we can create and train this custom model in MindsDB.

```sql
CREATE PREDICTOR mindsdb.byom_ray_serve_nlp
FROM maria (
    SELECT text, target
    FROM test.nlp_kaggle_train
) PREDICT target
USING
    url.train = 'http://127.0.0.1:8000/nlp_kaggle_model/train',
    url.predict = 'http://127.0.0.1:8000/nlp_kaggle_model/predict',
    dtype_dict={"text": "rich_text", "target": "integer"},
    format='ray_server';
```

The training process takes some time, considering that this model is a neural network rather than a simple logistic regression.

You can check the model status using this query:

```sql
SELECT *
FROM mindsdb.predictors
WHERE name='byom_ray_serve_nlp';
```

Once the status of the predictor has a value of `trained`, you can fetch predictions using the standard MindsDB syntax. Follow the guide on the [`SELECT`](/sql/api/select/) statement to learn more.

```sql
SELECT *
FROM mindsdb.byom_ray_serve_nlp
WHERE text='The tsunami is coming, seek high ground';
```

The expected output of the query above is `1`.

```sql
SELECT *
FROM mindsdb.byom_ray_serve_nlp
WHERE text='This is lovely dear friend';
```

The expected output of the query above is `0`.

!!! TIP "Wrong Results?"
    If your results do not match this example, try training the model for a longer amount of epochs.
