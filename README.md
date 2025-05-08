## start website django

```bash
python manage.py runserver
```

## start website with gunicorn

1. Log in to the Django admin with superuser credentials at `/admin/`.
2. Create or choose a user, and under the "Token" section, generate a token for that user.
3. Copy the generated token key.
4. Use the token in `Authorization: TOKEN <token>` for your requests.
   Alternative: Install the `authtoken` app and use a management command to generate a token.

After explaining the steps to generate the token, I’ll also provide instructions for using wget commands with the token.

## 1  Create / locate a token (one‑time)

> Turkle uses Django REST‑framework’s *token* authentication.
> The “key” lives in **Django‑admin** (`/admin/`) and belongs to a single user.

### 1.1  Make sure the token tables exist

```bash
# only needed the first time you turned on rest_framework.authtoken
python manage.py migrate authtoken
```

### 1.2  If you do **not** yet have a super‑user

```bash
python manage.py createsuperuser
# follow the prompts (username, email, password)
```

### 1.3  Log into the admin site

1. Open [http://127.0.0.1:6000/admin/](http://127.0.0.1:6000/admin/) in your browser.
2. Sign in with the super‑user credentials.

### 1.4  Generate (or view) the token

*In the left column you should now see **“Tokens”** (comes from `rest_framework.authtoken`).*

1. Click **Tokens → Add**.
2. Pick the user who will own the key (e.g. the super‑user or a new “service” user).
3. Click **Save** – Django shows you a 40‑character string (e.g. `4bcd5aa3...`).
   **Copy that key** – you will not see it again.

> **CLI shortcut**
> If you prefer the shell, run
>
> ```bash
> python manage.py drf_create_token <username>
> ```
>
> and Django prints the exact same 40‑char key.

---

## 2  Use the token in `wget`

Put the key in a shell variable so you can paste the following commands verbatim:

```bash
TOKEN="22baf699a21ef4a65c469e618baf2b3f1aa60420"   # 40‑char key from step 1.4
API="http://127.0.0.1:6000/api"
AUTH="Authorization: TOKEN $TOKEN"
```

---

## 3  REST calls – step by step

Below we create **two demo users**, **one project**, and **one batch** that shows a video, a question, and three multiple‑choice answers where each answer is flagged *correct* (`1`) or *incorrect* (`0`).

### 3.1  Add users

```bash
# Alice (id:3)
wget --method=POST -O - \
  --header="$AUTH" --header='Content-Type: application/json' \
  --body-data='{
     "username":"alice","password":"S3cretPwd!","first_name":"Alice","last_name":"Annotator"
  }'  "$API/users/"

# Bob (id:4)
wget --method=POST -O - \
  --header="$AUTH" --header='Content-Type: application/json' \
  --body-data='{
     "username":"bob","password":"S3cretPwd!","first_name":"Bob","last_name":"Checker"
  }'  "$API/users/"
```

Copy the `"id"` field in each JSON reply if you need it later.

### 3.2  Make a project (HTML template in one line for JSON)

```bash
PROJECT_HTML='<html><body><video width="640" controls>
<source src="{{ video_url }}" type="video/mp4"></video>
<p>{{ question_text }}</p>
<label><input type="radio" name="answer" value="A" data-correct="{{ answer_1_correct }}"> {{ answer_1 }}</label><br>
<label><input type="radio" name="answer" value="B" data-correct="{{ answer_2_correct }}"> {{ answer_2 }}</label><br>
<label><input type="radio" name="answer" value="C" data-correct="{{ answer_3_correct }}"> {{ answer_3 }}</label>
</body></html>'

wget --method=POST -O - \
  --header="$AUTH" --header='Content-Type: application/json' \
  --body-data='{
    "name":"Video‑QA Demo",
    "html_template":"'"${PROJECT_HTML//\"/\\\"}"'",
    "filename":"video_qa.html"
  }'  "$API/projects/"
```

Note the `"id"` in the response (say it is **5** → set `PROJECT_ID=5`).

### 3.3  Publish a batch of tasks

```bash
PROJECT_ID=5   # ← replace with real value from 3.2

CSV_TEXT='video_url,question_text,answer_1,answer_1_correct,answer_2,answer_2_correct,answer_3,answer_3_correct\n\
https://ia1.wse.jhu.edu/media/demo1.mp4,Which animal is shown?,Cat,0,Dog,1,Bird,0\n\
https://ia1.wse.jhu.edu/media/demo2.mp4,What colour is the car?,Red,0,Blue,1,Yellow,0'

wget --method=POST -O - \
  --header="$AUTH" --header='Content-Type: application/json' \
  --body-data='{
     "name":"Demo Batch",
     "project":'"$PROJECT_ID"',
     "filename":"video_qa.csv",
     "csv_text":"'"${CSV_TEXT//\"/\\\"}"'"
  }'  "$API/batches/"
```

The JSON reply includes the new batch’s `"id"`; use it for progress or results:

```bash
BATCH_ID=3   # example
wget -O results.csv --header="$AUTH" "$API/batches/$BATCH_ID/results/"
```

---

### Recap

1. **Token once** via Django‑admin (or `drf_create_token`).
2. Every API call adds `Authorization: TOKEN <key>`.
3. `wget` with `--method`, `--body-data`, and `Content-Type: application/json` lets you POST JSON straight from the shell.

You now have a working Turkle instance with users, a video‑QA project, and a populated batch ready for annotation.
