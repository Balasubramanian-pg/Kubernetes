# The one distinction that matters

Kubernetes orchestrates containers and their resources. It does **not** replace:

- Spark's distributed execution model
- Airflow's workflow orchestration
- A lakehouse's storage & table formats

> K8s = container scheduler + resource manager + service discovery.
> Everything else is Spark's job, Airflow's job, or the storage layer's job.
