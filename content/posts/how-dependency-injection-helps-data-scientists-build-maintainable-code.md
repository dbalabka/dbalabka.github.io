---
title: "How Dependency Injection Can Help Data Scientists Build Maintainable Code"
date: 2026-09-09T23:30:00+03:00
tags: ["Python", "Data Science", "Machine Learning", "Dependency Injection", "Architecture", "Jupyter", "12-Factor", "Dask"]
draft: false
---

Jupyter Notebooks are one of the most powerful tools in modern data science. They allow us to interactively explore raw data, quickly experiment with new algorithms, tweak hyperparameters on the fly, and visualize distributions and metrics within milliseconds. 

However, as data science teams scale and machine learning models move toward production, notebooks often become the source of significant technical debt. 

When working with data science teams across various organizations, I frequently observe notebooks plagued with hundreds of lines of boilerplate setup code that directly violate the [Twelve-Factor App](https://12factor.net/) methodology. Complex ML training pipelines that should easily run both locally in a notebook and unattended in a production Kubernetes cluster end up tangled in copy-paste cycles.

In my previous post, [Python Project Dependency Injection: Best Practices with python-inject](/posts/python-project-dependency-injection/), I explored how Dependency Injection (DI) and the lightweight library [`inject`](https://github.com/ivankorobkov/python-inject) bring architectural rigor and maintainability to enterprise Python web applications and CLIs. 

In this article, we will examine how **Dependency Injection can transform Data Science workflows**: allowing you to use the exact same environments, configurations, and core package code across interactive Jupyter Notebooks, production Kubernetes jobs, and distributed Dask clusters—without copying a single line of boilerplate.

---

## The Core Problem: The Notebook Copy-Paste Loop

Consider a typical machine learning project lifecycle:

1. **Exploration & Prototyping:** A data scientist opens a new Jupyter notebook, writes database queries, connects to a data warehouse (PostgreSQL, Snowflake, BigQuery), transforms features with Pandas, and trains a model prototype.
2. **Transitioning to Production:** To run this pipeline automatically (e.g., in a scheduled Airflow DAG, Kubeflow pipeline, or Docker container running a CLI command), software engineers or ML engineers extract that code into a structured Python package.
3. **The Divergence:** Two weeks later, model performance drifts in production. The data scientist needs to investigate the behavior, adjust hyperparameters, and visualize intermediate feature distributions.

What happens now?

In most teams, the engineer or data scientist **copies code out of the production package back into a notebook**. 

```
┌───────────────────────────────┐         Copy-paste         ┌───────────────────────────────┐
│     Jupyter Notebook          │ ─────────────────────────> │   Production Package (CLI)    │
│  - Raw DB Connection Strings  │                            │  - Config via ENVs            │
│  - Ad-hoc Class Construction  │ <───────────────────────── │  - Structured Modules         │
│  - Scratchpad Pipelines       │      Manual extraction     │  - Container Entrypoints      │
└───────────────────────────────┘                            └───────────────────────────────┘
```

They copy the database connection logic, instantiate several service classes, replicate preprocessing steps, and manually wire dependencies just to reproduce a production run. If the production code changes (such as a database schema update, connection pool adjustment, or new logging handler), the notebook silently becomes obsolete.

### Why Papermill Is Not a Silver Bullet

Some teams attempt to solve this by parameterizing notebooks using tools like [Papermill](https://papermill.readthedocs.io/). While Papermill is great for executing linear reporting notebooks on a schedule, running notebooks as your primary production application violates fundamental software engineering practices:

- Notebook JSON files (`.ipynb`) are notoriously difficult to diff, code review, and version control.
- Hardcoded infrastructure and connection logic inside cells makes unit testing individual units of business logic nearly impossible.
- Reproducibility across multiple environments (local vs. staging vs. Kubernetes) remains fragile.

To build genuinely maintainable ML systems, we need a fundamental shift in how we view notebooks.

---

## The Architectural Shift: The Notebook is Just an Entry Point

In robust software architecture, **your notebook should not be your codebase**. 

Instead, treat your Jupyter Notebook as **just another entry point** into your core Python package—on equal footing with a Typer CLI command, a FastAPI web endpoint, or an Airflow task runner:

```
                          ┌────────────────────────┐
                          │   Core Python Package  │
                          │      (Domain Logic)    │
                          │   - Repositories       │
                          │   - Model Pipelines    │
                          │   - Preprocessors      │
                          └───────────▲────────────┘
                                      │
                         Glued together by di.py
                                      │
         ┌────────────────────────────┼────────────────────────────┐
         │                            │                            │
┌────────┴────────┐          ┌────────┴────────┐          ┌────────┴────────┐
│ Jupyter Notebook│          │  Production CLI │          │   Distributed   │
│  (Interactive)  │          │ (Docker / K8s)  │          │  Dask Workers   │
└─────────────────┘          └─────────────────┘          └─────────────────┘
```

When you adopt this mental model:
- All domain logic, feature transformations, repositories, and model training routines reside in your version-controlled, testable package (e.g., `src/ml_project/`).
- Notebooks contain **only** high-level calls, exploratory code, visualizations, and interactive plots.
- Running a full training pipeline or fetching data in a notebook should require **one single line of code**.

To achieve this without duplicating setup code across entry points, we need a clean way to assemble our classes and their configurations.

---

## The Challenge: Twelve-Factor Configuration & Infrastructure Leakage

Let's look at a concrete example: fetching training data from a database and visualizing it in a notebook.

According to the [Twelve-Factor App methodology (Factor III: Config)](https://12factor.net/config), configuration—especially sensitive credentials like database passwords, API tokens, and hostnames—must be strictly separated from code and stored in environment variables.

### The Naive Notebook Approach

Data scientists often load credentials into a notebook like this:

```python
# ❌ Anti-pattern: Leaking infrastructure setup and credentials into notebook cells
import os
from dotenv import load_dotenv
from sqlalchemy import create_engine
import pandas as pd

load_dotenv()

# Hardcoded or copied connection wiring
db_user = os.getenv("DB_USER", "postgres")
db_pass = os.getenv("DB_PASSWORD", "secret")
db_host = os.getenv("DB_HOST", "localhost")
db_name = os.getenv("DB_NAME", "analytics")

DATABASE_URL = f"postgresql://{db_user}:{db_pass}@{db_host}:5432/{db_name}"

# Manually configuring engine and pools inside notebook
engine = create_engine(DATABASE_URL, pool_size=5, pool_pre_ping=True)

query = "SELECT user_id, signup_date, total_spend FROM customer_features"
df = pd.read_sql(query, con=engine)
```

### Why this is problematic:
1. **Security Risks:** Credentials easily get copied into cells, checked into Git repositories, or rendered into notebook cell outputs.
2. **Duplicated Plumbing:** If three data scientists work on different analyses, all three duplicate the `create_engine` parameters, connection timeout settings, and query logic.
3. **Lack of Abstraction:** If the data access layer migrates from PostgreSQL to Snowflake or adds caching and logging, every single notebook breaks.

---

## Step 1: The Factory Pattern and Its Limits

A natural software engineering instinct is to encapsulate the connection creation into a reusable factory function inside your package:

```python
# ml_project/infrastructure/db.py
import os
from sqlalchemy import create_engine, Engine

def create_db_engine() -> Engine:
    db_url = os.getenv("DATABASE_URL")
    if not db_url:
        raise ValueError("DATABASE_URL environment variable is not set")
    return create_engine(db_url, pool_size=10, pool_pre_ping=True)
```

And a factory for your repository:

```python
# ml_project/repository.py
from logging import Logger
from sqlalchemy import Engine
import pandas as pd

class CustomerRepository:
    def __init__(self, engine: Engine, logger: Logger):
        self.engine = engine
        self.logger = logger

    def fetch_features(self) -> pd.DataFrame:
        self.logger.info("Fetching customer feature data...")
        return pd.read_sql("SELECT * FROM customer_features", con=self.engine)

def create_customer_repository(logger: Logger) -> CustomerRepository:
    engine = create_db_engine()
    return CustomerRepository(engine=engine, logger=logger)
```

### The Drawbacks of Pure Factories in Complex ML Pipelines

Factories are an improvement, but as machine learning systems grow, factories run into severe scalability bottlenecks:

1. **Parameter Drilling & Nested Factory Hell:** An ML training pipeline rarely depends on just a database. It depends on an `ExperimentTracker` (e.g. MLflow/WandB), a `FeatureStoreClient`, a `ModelRegistry`, cloud storage (`S3Client`), and custom preprocessors. A top-level factory `create_training_pipeline()` must either know how to instantiate every sub-dependency or take dozens of arguments and drill them down several layers deep.
2. **Rigid Lifecycles:** If you want to reuse a shared database connection pool or singleton logger across multiple components, managing their lifecycles across multiple custom factory functions becomes tedious and bug-prone.
3. **Difficulty Overriding for Local Experiments:** What if you want to run the full training pipeline in your notebook, but swap the production database with a lightweight local SQLite database or in-memory sample? With hardcoded factories, you must modify the factory code or pass override flags everywhere.

This is precisely the problem that **Dependency Injection (DI) containers** solve.

---

## The Solution: Dependency Injection with `python-inject`

A Dependency Injection container (or Service Container) standardizes object instantiation and lifecycle management. It acts as an inversion-of-control engine: classes declare what they need in their constructors, and the container automatically resolves and provides those dependencies at runtime.

In Python, [`inject`](https://github.com/ivankorobkov/python-inject) is an ideal choice for Data Science projects because:
- It is lightweight, non-invasive, and thread-safe.
- It does not force you to inherit from base classes or clutter your domain logic with framework code.
- It enables complete environment parity across notebooks, CLIs, and distributed workers.

Let’s see how to implement this architecture step by step.

---

### Step 1: Install the Library

Add `inject` and `python-dotenv` to your project dependencies:

```bash
pip install inject python-dotenv typer
```

Or using Poetry:

```bash
poetry add inject python-dotenv typer
```

---

### Step 2: Organize Your Project Structure

A clean, maintainable project structure separates your core package from its various entry points:

```text
my_ml_project/
├── .env                       # Local environment variables (git-ignored)
├── notebooks/
│   └── customer_churn_eda.ipynb # Interactive notebook entry point
├── src/
│   └── ml_project/
│       ├── __init__.py
│       ├── config.py          # 12-factor settings
│       ├── di.py              # Composition Root (central container config)
│       ├── infrastructure/
│       │   ├── db.py          # Engine & connection helpers
│       │   └── logging.py     # Application logger
│       ├── repository.py      # Data access layer
│       └── pipeline.py        # ML training & evaluation logic
└── cli.py                     # CLI entry point (Typer for Docker/K8s)
```

---

### Step 3: Write Pure Domain Classes (POPOs)

Keep your data access classes and training pipelines completely free of DI decorators or framework imports. They should be **POPOs (Plain Old Python Objects)** that declare their dependencies via type-annotated `__init__` parameters:

```python
# src/ml_project/repository.py
import pandas as pd
from sqlalchemy import Engine
from logging import Logger

class CustomerRepository:
    """Pure domain repository. Zero knowledge of inject or di.py."""
    def __init__(self, engine: Engine, logger: Logger):
        self.engine = engine
        self.logger = logger

    def fetch_training_data(self) -> pd.DataFrame:
        self.logger.info("Extracting training dataset from database...")
        query = "SELECT user_id, age, tenure, monthly_charges, churn FROM customer_features"
        return pd.read_sql(query, con=self.engine)
```

```python
# src/ml_project/pipeline.py
from logging import Logger
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from ml_project.repository import CustomerRepository

class ModelTrainingPipeline:
    """Pure pipeline orchestrator."""
    def __init__(self, repository: CustomerRepository, logger: Logger):
        self.repository = repository
        self.logger = logger

    def run(self, n_estimators: int = 100, max_depth: int = 5) -> RandomForestClassifier:
        self.logger.info(f"Starting training run (trees={n_estimators}, depth={max_depth})")
        df = self.repository.fetch_training_data()
        
        X = df[["age", "tenure", "monthly_charges"]]
        y = df["churn"]

        model = RandomForestClassifier(n_estimators=n_estimators, max_depth=max_depth, random_state=42)
        model.fit(X, y)
        self.logger.info("Training successfully completed.")
        return model
```

---

### Step 4: Centralize Configuration in `di.py` (Composition Root)

The `di.py` module is your application's **Composition Root**. It connects your classes with their dependencies and configuration loaded from environment variables:

```python
# src/ml_project/di.py
import os
import inject
from logging import Logger, getLogger, StreamHandler, Formatter
from sqlalchemy import create_engine, Engine
from ml_project.repository import CustomerRepository
from ml_project.pipeline import ModelTrainingPipeline

# Re-export inject so entry points can import it directly
__all__ = ["inject"]


def provide_logger() -> Logger:
    logger = getLogger("ml_project")
    if not logger.handlers:
        logger.setLevel(os.getenv("LOG_LEVEL", "INFO"))
        handler = StreamHandler()
        handler.setFormatter(
            Formatter("[%(asctime)s] [%(levelname)s] %(name)s: %(message)s")
        )
        logger.addHandler(handler)
    return logger


def provide_db_engine() -> Engine:
    db_url = os.getenv("DATABASE_URL", "sqlite:///sample.db")
    return create_engine(db_url, pool_size=5, pool_pre_ping=True)


@inject.autoparams()
def provide_customer_repository(engine: Engine, logger: Logger) -> CustomerRepository:
    return CustomerRepository(engine=engine, logger=logger)


@inject.autoparams()
def provide_training_pipeline(
    repository: CustomerRepository, 
    logger: Logger
) -> ModelTrainingPipeline:
    return ModelTrainingPipeline(repository=repository, logger=logger)


def configure(binder: inject.Binder) -> None:
    # 1. Bind singletons and infrastructure providers
    binder.bind_to_constructor(Logger, provide_logger)
    binder.bind_to_constructor(Engine, provide_db_engine)

    # 2. Bind domain services & pipelines
    binder.bind_to_constructor(CustomerRepository, provide_customer_repository)
    binder.bind_to_constructor(ModelTrainingPipeline, provide_training_pipeline)


# Self-initialize the container upon module import
inject.configure(configure, once=True, bind_in_runtime=False)
```

#### Why this pattern shines:
- **Automatic Parameter Resolution (`@inject.autoparams`):** Notice that `provide_training_pipeline` simply asks for `CustomerRepository` and `Logger`. The container inspects the type annotations and wires them automatically.
- **`once=True`:** Ensures that reloading cells in a Jupyter Notebook does not raise a `BindingException`.
- **Environment Compliance:** The database engine seamlessly consumes `DATABASE_URL` from the environment, satisfying the Twelve-Factor App standard.

---

## Step 5: Consuming in Entry Points

Now comes the payoff. Look at how clean and uncluttered our different entry points become.

### Entry Point A: Inside the Jupyter Notebook

In your notebook (`notebooks/customer_churn_eda.ipynb`), you no longer write connection strings, SQL client initialization, or class wiring. You load your local `.env` and request what you need from the container in **one line**:

```python
# Cell 1: Setup environment
from dotenv import load_dotenv
load_dotenv("../.env")

from ml_project.di import inject
from ml_project.repository import CustomerRepository
from ml_project.pipeline import ModelTrainingPipeline
```

```python
# Cell 2: Fetch and visualize data interactively
repo = inject.instance(CustomerRepository)
df = repo.fetch_training_data()

# Instant EDA without copy-pasted DB boilerplate!
display(df.head())
df["monthly_charges"].plot(kind="hist", title="Monthly Charges Distribution")
```

```python
# Cell 3: Run the production pipeline with custom hyperparameters
pipeline = inject.instance(ModelTrainingPipeline)
trained_model = pipeline.run(n_estimators=250, max_depth=8)
```

Notice what just happened:
- If someone updates the SQL query or optimizes the database connection pool in the core package, your notebook **automatically benefits from the update on the next run**.
- Zero credentials or connection setup code are saved in the notebook file.
- You can experiment with model hyperparameter tuning interactively while executing the exact code that runs in production.

---

### Entry Point B: Production CLI with Typer

In production (e.g., inside a Docker container orchestrated by a Kubernetes CronJob or Airflow task), the entry point is a clean CLI built with [Typer](https://typer.tiangolo.com/):

```python
# cli.py
import typer
from dotenv import load_dotenv

# Load env variables if running outside container
load_dotenv()

from ml_project.di import inject
from ml_project.pipeline import ModelTrainingPipeline

app = typer.Typer(help="ML Production Training CLI")


@app.command()
def train(n_estimators: int = 100, max_depth: int = 5):
    """Production entry point: run the model training pipeline."""
    pipeline = inject.instance(ModelTrainingPipeline)
    pipeline.run(n_estimators=n_estimators, max_depth=max_depth)
    typer.echo("Pipeline completed successfully.")


if __name__ == "__main__":
    app()
```

Both the Jupyter notebook and the production CLI execute the identical `ModelTrainingPipeline` with the exact same dependencies—guaranteeing **complete development-to-production parity**.

---

## Real-World Distributed Scale: Dask Cluster Integration

The benefits of Dependency Injection become even more apparent when moving beyond a single machine to **distributed computing with Dask**.

In distributed machine learning, data scientists often use a Jupyter Notebook to coordinate heavy distributed workloads across a cluster of remote cloud worker machines (e.g., on Google Cloud or AWS).

A frequent obstacle is distributing custom local package modules to the Dask workers without rebuilding and redeploying entire Docker images for every experimental iteration. 

In my demo project [`dbalabka/dask-module-upload-plugin-demo`](https://github.com/dbalabka/dask-module-upload-plugin-demo), I showcased a solution to this problem using a custom Dask upload plugin (proposed in [dask/distributed#8884](https://github.com/dask/distributed/pull/8884)).

Look at how seamlessly Dependency Injection works across a distributed cluster:

```python
# Inside a Jupyter notebook connected to a remote GCP Dask Cluster
from dask.distributed import Client
from dask_cloudprovider.gcp import GCPCluster
import dask

# Connect to cloud Dask cluster
cluster = GCPCluster(projectid="my-gcp-project", zone="us-central1-a", n_workers=5)
client = Client(cluster)

# Upload the local package to all remote workers on the fly
from dask_module_upload_plugin_demo.plugins import UploadModule, SchedulerUploadModule
import ml_project

client.register_plugin(UploadModule(ml_project))
client.register_plugin(SchedulerUploadModule(ml_project))
```

Now, when executing a delayed function on a remote Dask worker node, the worker can simply resolve its dependencies through `inject`:

```python
def distributed_data_processing():
    """Runs remotely inside a Dask worker pod/VM."""
    from ml_project.di import inject
    from ml_project.repository import CustomerRepository

    # Resolve fully configured repository on the remote worker
    repo = inject.instance(CustomerRepository)
    return repo.fetch_training_data()

# Compute remotely across the cluster
future = dask.delayed(distributed_data_processing)()
remote_dataframe = future.compute()
```

Because the DI configuration in `di.py` encapsulates all construction logic, remote workers execute the task using their own local environment variables and connection pools with zero manual initialization.

---

## Effortless Testing and Experiment Overrides

Another huge advantage of DI for data scientists is the ability to swap components for local testing or benchmarking without modifying any source code.

Suppose you want to run your training pipeline in a continuous integration (CI) test suite without connecting to a live database:

```python
# tests/test_pipeline.py
import pytest
import inject
import pandas as pd
from unittest.mock import MagicMock
from sqlalchemy import Engine
from logging import Logger
from ml_project.repository import CustomerRepository
from ml_project.pipeline import ModelTrainingPipeline


class FakeCustomerRepository(CustomerRepository):
    """Test double returning in-memory synthetic data."""
    def __init__(self):
        pass

    def fetch_training_data(self) -> pd.DataFrame:
        return pd.DataFrame({
            "user_id": [1, 2, 3],
            "age": [25, 45, 35],
            "tenure": [12, 48, 2],
            "monthly_charges": [50.0, 95.5, 30.0],
            "churn": [0, 1, 0]
        })


def test_training_pipeline_with_mock_repository():
    # 1. Pure unit test without DI:
    fake_repo = FakeCustomerRepository()
    mock_logger = MagicMock(spec=Logger)
    
    pipeline = ModelTrainingPipeline(repository=fake_repo, logger=mock_logger)
    model = pipeline.run(n_estimators=10, max_depth=2)
    
    assert model is not None
    assert hasattr(model, "predict")
```

And in integration fixtures, you can rebind the container on the fly using `inject.clear_and_configure()`:

```python
# tests/conftest.py
import pytest
import inject
from ml_project.repository import CustomerRepository
from tests.test_pipeline import FakeCustomerRepository

@pytest.fixture(autouse=True)
def setup_test_di():
    def test_config(binder: inject.Binder):
        binder.bind(CustomerRepository, FakeCustomerRepository())

    inject.clear_and_configure(test_config)
    yield
    inject.clear()
```

---

## Conclusion: Clean Code for Serious Data Science

Data science and machine learning systems do not have to be messy. The perception that Jupyter notebooks inherently lead to sloppy, unmaintainable code is false—the problem lies in treating notebooks as full applications rather than entry points.

By introducing Dependency Injection with `python-inject`:

1. **You eliminate the copy-paste cycle:** Domain logic, data access, and training pipelines live once in your version-controlled package.
2. **You adhere to the Twelve-Factor App:** Credentials and configurations are loaded from environment variables and resolved cleanly through the container.
3. **Notebooks become lean and powerful:** Exploring data or running training runs requires just one line of code: `inject.instance(...)`.
4. **Parity is guaranteed:** Your interactive experiments, Kubernetes CLI jobs, CI test suites, and distributed Dask clusters all run the exact same underlying code.

If you are ready to take your data science codebase to the next level, start by extracting your database wiring into a centralized `di.py`. Your future self—and your engineering team—will thank you.
