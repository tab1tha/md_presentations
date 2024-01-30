---
marp: true
---
**Creating a validator**
You need a way to
- keep track of the rules that exist.
- keep track of the failing locations that have been found per rule.
- modify the ruleset over time.
- step back into previous versions of the ruleset.

---
But first, Poetry

For dependency management and environment isolation
```poetry init``` at the start.
```poetry install``` every time after that.
+++++
```poetry add <package_name>```
```poetry shell``` creates the environment each time.

---
**Sample**: R100 <ReferenceDate> (N00603) must be present and must equal 2022-03-31
```python
@rule_definition(
    code="100",
    module=CINTable.Header,
    message="Reference Date is incorrect",
    affected_fields=[ReferenceDate],
)
def validate(
    data_container: Mapping[CINTable, pd.DataFrame], rule_context: RuleContext
):
    df = data_container[Header]

    # ReferenceDate is expected to be the 31st of March in the collection year
    df["expected_date"] = f"31/03/{df[Year]}"
    df["expected_date"] = pd.to_datetime(
        df["expected_date"], format="%d/%m/%Y", errors="coerce"
    )

    is_present = df[ReferenceDate].isna()
    error_date = df[ReferenceDate] != df["expected_date"]
    failing_indices = df[is_present | error_date].index

    rule_context.push_issue(table=Header, field=ReferenceDate, row=failing_indices)
```

---
**Keeping track of the rules that exist**

@rule_definition decorator assigns marker
```python
wrapper.__rule_def__ = RuleDefinition(code, func, rule_type ...)
```
init.py of ruleset pulls in filepaths
```python
files = Path(__file__).parent.glob("*.py")
```

At runtime, marker is used as filter
```python
rule_content = importlib.import_module(f"{path.stem}")
validator_func = {
            str(element.__rule_def__.code): element.__rule_def__
            for _, element in vars(rule_content).items()
            if hasattr(element, "__rule_def__")
        }
```
---
**Keep track of failing locations per rule**
In validation function
```python
rule_context.push_type_1(
        table=CINdetails, columns=[ReferralNFA, PrimaryNeedCode], row_df=df_issues
    )
```
In central module
```python
class RuleContext:
    def push_type_1(self, table, columns, row_df):
        self.__type1_issues = Type1(table, columns, row_df)
```
where
```python
class Type1:
    table: CINTable
    columns: List[str]
    row_df: pd.DataFrame
```
---
**Modifying the ruleset over time**
Create object that describes how previous ruleset should be modified when run.
```python
del_list: list[str] = []
this_year_config = YearConfig(
    deleted=del_list, added_or_modified=this_year_validator_funcs
)

registry = update_validator_functions(prev_registry, this_year_config)
```
Dictionary methods are used for updating ruleset
```python
updated_validator_funcs = prev_validator_funcs | this_year_config.added_or_modified

for deleted_rule in this_year_config.deleted:
    del updated_validator_funcs[deleted_rule]
```
---
**Step back into previous versions of ruleset**
```python
def get_year_ruleset(collection_year: str) -> dict[str, RuleDefinition]:
    """
    Gets the registry of validation rules for the year specified in the metadata.
    """
    # for example,"2023" is converted to "cin2022_23"
    ruleset = f"cin{int(collection_year)-1}_{collection_year[2:4]}"

    module = importlib.import_module(f"cin_validator.rules.{ruleset}")
    registry = getattr(module, "registry")

    return registry
```
used as 
```python
ruleset = get_year_ruleset(file_metadata["collectionYear"])
```

---
**Communicating with the frontend: prpc**
```python
from prpc_python import RpcApp
app = RpcApp("validate_cin")

@app.call
def cin_validate(
    cin_data: dict,
    file_metadata: dict,
    selected_rules: Optional[list[str]] = None,
):
    ruleset = get_year_ruleset(file_metadata["collectionYear"])
    validator = cin_validator.CinValidator(data_files, ruleset, selected_rules)

    # make return data json-serialisable
    issue_report = validator.full_issue_df.to_json(orient="records")
    # what the user will download
    user_report = validator.user_report.to_json(orient="records")

    validation_results = {
        "issue_locations": [issue_report],
        "user_report": [user_report],
    }
    return validation_results
```
---
**Separation of concerns**
api handler 
```python
import xml.etree.ElementTree as ET
filetext = cin_data_file.read().decode("utf-8")
root = ET.fromstring(filetext)
```
cli handler
```python
fulltree = ET.parse(filename)
root = fulltree.getroot()
```
then central module
```python
# generate tables from XML
data_files = ProcessData(root)
```

---
**Checking that it all works: Pytest, Click**
[Sample test: Rule 8610](https://github.com/data-to-insight/csc-validator-be-cin/blob/main/cin_validator/rules/cin2022_23/rule_8610.py)

```bash
python -m cin_validator test
```
where 
```python
test_files = [
            str(p.absolute())
            for p in module_folder.glob("*.py")
            if p.stem != "__init__"
        ]
pytest.main(test_files)
```
---
**Verify output to frontend**
```bash
prpc run -a rpc_main:app cin_validate
```
**Run full app from command line interface**
```bash
python -m cin_validator run <file_path>
```
In each case, outputs can be written to files and saved instead of being displayed on the command line.

---
**Operations: Git**
pre-commit hooks for autoformatting.
```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
    - id: black
  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
    - id: isort
      args: [ "--profile", "black" ]
``` 
---
**Operations: GitHub Actions**
Auto-publish to PyPI _(pseudocode)_
```yaml
deploy:
  - name: Publish package
      run: poetry publish
      env:
        POETRY_PYPI_TOKEN_PYPI: ${{ secrets.PYPI_API_TOKEN }}
``` 
Auto-test on push _(pseudocode)_
```yaml
build:
  - name: Build and run dev container task
        uses: devcontainers/ci@v0.2
        with:
          runCmd: python -m cin_validator test
``` 
---
**More things that helped**
Automatic issue creation per validation rule
```python
gh = Github(token)
repo = gh.get_repo("SocialFinanceDigitalLabs/CIN-validator")
repo.create_issue(title, body)
```
Keeping track of rule patterns.
```md
#### - value must be present and equal to ...
100, 4180, 4220, 8500, 8568, 8600, 8640, 8650, 8910, 8842Q

#### - there must be only one in group
4004, 8794, 8839, 8896, 8935, 8898, 8815
```

---
Plan
- poetry for dependency resolution and environment isolation.
- Show DfE guidance (Screenshot of excel rules)
- Introduce what it takes to build a validator. Show sample rule.
- Talk of registry. Show version control from year to year.
- Show unit tests. Q/A solution since data is absent.
- Click. Run on file. Run tests.
- Show rpc main.py. Refer to cinvalidator.py. Then show CLI handler.
- Show how file handling is decentralised. Refer to arc diagram
- Show how to test rpc part. emphasize modularisation.
- Show wheel and rpc init in frontend.
- workflows. Pre-commit, GA auto-testing, publish automation,autoissue
- Documentation. Rule_Patterns.md. managing env variables with config.
---

Tools
- Poetry
- Pandas
- importlib
- prpc
- Click
- Pytest
- Markdown
- Github actions: auto-testing and auto-publishing.
- Github hooks: Pre-commit
- Github api: issue auto-creation
- Django, decouple(config)
- React, TypeScript, Figma (UT)