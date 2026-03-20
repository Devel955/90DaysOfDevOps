# Day 40 - My First GitHub Actions Workflow

## My Workflow YAML
```yaml
name: Hello Workflow

on:
  push:

jobs:
  greet:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Print greeting
        run: echo "Hello from GitHub Actions!"

      - name: Print date and time
        run: date

      - name: Print branch name
        run: echo "Branch is $GITHUB_REF_NAME"

      - name: List files in repo
        run: ls -la

      - name: Print operating system
        run: echo "OS is $RUNNER_OS"
```

## Screenshot
![Green Pipeline Run](./pipeline-screenshot.png)

## What Each Key Does (My Own Words)

### `on:`
Yeh batata hai ki workflow KAB chalegi.
Humne `push` likha hai — matlab jab bhi main branch pe
code push hoga, pipeline automatically start ho jayegi.

### `jobs:`
Yeh batata hai ki KYA kaam karna hai.
Ek workflow mein kai jobs ho sakti hain.
Humari ek job hai jiska naam `greet` hai.

### `runs-on:`
Yeh batata hai ki job KIS machine pe chalegi.
Humne `ubuntu-latest` likha — matlab GitHub ek
fresh Linux computer deta hai har run ke liye.

### `steps:`
Job ke andar chhote chhote kaam hote hain.
Yeh upar se neeche ek ek karke chalte hain.
Agar ek step fail ho jaye toh pipeline wahan ruk jaati hai.

### `uses:`
Kisi ready-made action ko use karna.
`actions/checkout@v4` GitHub ka official tool hai
jo hamara code runner pe download karta hai.

### `run:`
Seedha terminal command chalana.
Jaise `echo`, `date`, `ls` — koi bhi bash command
yahan likh sakte hain.

### `name:` (step pe)
Step ka label hota hai — sirf dikhane ke liye.
Actions tab ke logs mein yahi naam dikhta hai
taaki pata chale kaunsa step chal raha hai.

## What I Learned Today

- GitHub Actions automatically trigger hoti hai push pe
- Har run ek fresh Ubuntu machine pe hota hai
- Steps upar se neeche chalte hain — ek fail = pipeline stop
- `exit 1` se deliberately pipeline tod sakte hain
- Logs mein har step ka output dekh sakte hain
- CI/CD ab sirf concept nahi — real hai!
