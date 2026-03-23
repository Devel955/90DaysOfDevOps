Day 43 – Jobs, Steps, Environment Variables & Conditionals
🚀 Why This Matters

In real-world CI/CD pipelines, it's not enough to just run commands.
We need to control execution flow, share data between jobs, and make decisions dynamically.

Today’s learning focused on making pipelines:

Structured (using multiple jobs)
Dynamic (using conditionals)
Connected (using outputs)
Configurable (using environment variables)

This is exactly how production-grade pipelines are designe

🧩 What I Built
🔹 Multi-Job Workflow
Created 3 jobs: build → test → deploy
Used needs: to control execution order
Learned that jobs run in parallel by default

📸 Screenshot:<img width="1920" height="965" alt="Screenshot 2026-03-23 214811" src="https://github.com/user-attachments/assets/0d51d25d-c90c-486d-ab58-574afd2ecba7" />

🔹 Environment Variables
Used variables at:
Workflow level
Job level
Step level
Accessed GitHub context:
${{ github.sha }}
${{ github.actor }}
📸 Screenshot:<img width="1920" height="968" alt="Screenshot 2026-03-23 215417" src="https://github.com/user-attachments/assets/c30ee90d-0f47-454f-99f4-0146e01774a8" />

🔹 Job Outputs
Generated data in one job (date)
Passed it to another job using outputs
Learned that jobs run on separate runners → no shared memory

📸 Screenshot:<img width="1920" height="965" alt="Screenshot 2026-03-23 221201" src="https://github.com/user-attachments/assets/3adefdfb-9d04-4c61-b044-f28942087ca1" />

🔹 Conditionals
Controlled execution using if:
Ran steps only on:
push events
main branch
Simulated failure using exit 1
Used continue-on-error to continue pipeline
📸 Screenshot:<img width="1920" height="962" alt="Screenshot 2026-03-23 222656" src="https://github.com/user-attachments/assets/ef93e31b-8422-4a69-9f67-0dd42d532f3d" />

🔹 Smart Pipeline
Ran lint and test in parallel
Used needs: to run summary after both
Printed:
Branch type (main vs feature)
Commit message
📸 Screenshot:<img width="1920" height="965" alt="Screenshot 2026-03-23 223048" src="https://github.com/user-attachments/assets/3aca8c2a-365f-4bf3-a99c-c3473dfc2fdd" />

🧠 Key Concepts Learned
✔ Jobs & Execution Flow
Jobs run in parallel by default

needs: creates dependency chain
✔ Environment Variables
Help manage configuration
Reduce duplication

Improve maintainability
✔ Outputs Between Jobs
Used to pass dynamic data

Required because jobs are isolated
✔ Conditionals
if: controls execution

Makes pipelines intelligent and dynamic
✔ continue-on-error
Allows failure without stopping pipeline
Useful for logging and recovery scenarios

🎯 Real-World Understanding
  Today I realized that:
  
CI/CD is not just automation
It is about decision-making, control, and flexibility
A good pipeline:

Runs tasks efficiently
Handles failures smartly
Adapts based on conditions
💼 Interview Takeaway

👉
"GitHub Actions workflows can be designed using jobs, environment variables, outputs, and conditionals to create dynamic and production-ready CI/CD pipelines."

📌 Conclusion
Day 43 helped me move from:
➡ simple pipelines
➡ to smart, production-level workflows

This is a major step toward becoming a DevOps engineer.
#90DaysOfDevOps #DevOpsKaJosh #TrainWithShubham




