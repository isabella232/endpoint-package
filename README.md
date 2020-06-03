# The Endpoint Package

The Endpoint package handles installing mapping templates, ILM policies, ingest pipelines, and other functionality
that should be done prior to the Endpoint streaming data to ES and using the Endpoint app in Kibana.

The Endpoint package is located [locally here](./package/endpoint) and [remotely here](https://github.com/elastic/package-storage/tree/master/packages/endpoint)

To update the endpoint package clone this repo and make changes as needed

## Tool Prerequisites

This section includes a list of tools that should be installed before making changes to the Endpoint's mapping.
The individual sections below will give more details about how each tool is used and setup.

### Package Registry

- Install go 1.13 from here: <https://golang.org/dl/>

- Install [mage](https://github.com/magefile/mage#installation)

### Mapping Generator Scripts

If you're going to use the Makefile, install pipenv, python and the individual packages will be installed for you

If you're planning on running the commands yourself:

- Install Python 3.7+

- Clone the ecs repo: <https://github.com/elastic/ecs>

- Install the requirements.txt from here: <https://github.com/elastic/ecs/blob/master/scripts/requirements.txt>

## Testing Changes

To test changes to the Endpoint package you will need to point your Kibana at a locally running package registry. To
run the package registry follow the instructions in the [package registry](https://github.com/elastic/package-registry/blob/master/README.md#running)

Here are the distilled steps:

- Install go 1.13 from here: <https://golang.org/dl/>

Use the `make run-registry` command to quickly run a package registry locally once you have go installed.

Add the follow flags to your `kibana.dev.yaml` file

```yaml
xpack.ingestManager.enabled: true
xpack.ingestManager.epm.registryUrl: "http://127.0.0.1:8080"
xpack.ingestManager.epm.enabled: true
xpack.ingestManager.fleet.enabled: true
```

The `xpack.ingestManager.epm.registryUrl` flag instructs Kibana to look for the package registry at the specified URL.
By default Kibana uses the external package registry.

The Ingest Manager will now use your locally running package registry for retrieving a package. The Ingest Manager
within Kibana does some caching after it has downloaded a package, so if you are not seeing your changes you might
need to restart Kibana and Elasticsearch.

## Updating the Endpoint Package Mapping

To update the endpoint package mapping take a look at the [Custom Schema](./custom_schemas/README.md) and
[Custom Subset](./custom_subsets/README.md) first to get an understanding of what makes up the mapping.

The essential steps are listed here:

- Edit/add custom schema files as needed to define any fields that don't exist in ECS core

- Update the subset files for the particular event that is being changed or create a new subset file

- If a new type of document is being added that doesn't fit in the existing ones (e.g. events),
  create a new directory in `custom_subsets/elastic_endpoint` to contain the subset files

- Generate the mapping

## ECS Version

The generated files are dependent on the github version of ECS used. To use a more recent version
of ECS to pick up new definition chance the `ECS_GIT_REF` in the **Makefile**. You can also
make a temporary change command line `make ECS_GIT_REF=v1.6.0`. But be sure to commit this to the
**Makefile** when you are done and satisfied with your change.

## Generating the files with the Makefile

First create a file called `config.mk` with the location of your ecs repo. Here's an example:

```makefile
ECS_DIR := ../ecs
```

You will need pipenv installed: <https://pipenv-fork.readthedocs.io/en/latest/install.html>

Then run `make`

The created files will be in the directory `out`. The makefile takes care of formatting
the file as described below.

## Generating the files manually

```bash
cd ecs
python scripts/generator.py --out ../<some directory to write the generated files> --include ../endpoint-app-team/custom_schemas --subset ../endpoint-app-team/custom_subsets/<subset file(s) being used>
```

Here are some examples:

### Generating the events mapping

```bash
python scripts/generator.py --out ../gen --include ../endpoint-app-team/custom_schemas --subset ../endpoint-app-team/custom_subsets/elastic_endpoint/events/*
```

### Generating the metadata mapping

```bash
python scripts/generator.py --out ../gen --include ../endpoint-app-team/custom_schemas --subset ../endpoint-app-team/custom_subsets/elastic_endpoint/metadata/*
```

The generated files will be in `../gen` in the examples above. The file needed to update the mapping is in
`../gen/generated/beats/fields.ecs.yml`

### Formatting the file

The file format needs to be updated slightly before being included in the package. To get the file in the right format
remove these lines:

```yaml
# WARNING! Do not edit this file directly, it was generated by the ECS project,
# based on ECS version 1.5.0-dev.
# Please visit https://github.com/elastic/ecs to suggest changes to ECS fields.

- key: ecs
  title: ECS
  description: ECS Fields.
  fields:
```

Next, remove the indentation.

Now copy this file to <./package/endpoint/dataset/events/fields/fields.yml>
if you are updating the `events` mapping. If you are updating the `metadata`, the `metadata` file is here: <./package/endpoint/dataset/metadata/fields/fields.yml>

Rerun `mage build` and `go run .` (or just `make run-registry`) and the endpoint package should have the latest mapping changes in it. You will probably
have to restart ES and Kibana because the Ingest Manager keeps the installed packages in a cache so it might not try to
pull down the latest one (upgrading a package hasn't been implemented as of 4/9/20 when writing this README).

## Updating the package in staging

Once testing is complete, PR and merge your schema changes to the package to this repo.
To release a new endpoint package a PR will be opened against the package-storage repo <https://github.com/elastic/package-storage> with
the contents of the new endpoint package. A new version number directory should be created here: <https://github.com/elastic/package-storage/tree/master/packages/endpoint> with the appropriate version number for this release. Once this PR is merged a docker image will be built containing
the new endpoint package. There is still a manual step of pushing the docker image to the staging environment that is currently done
by the Ingest Management team.

Once the Endpoint team is given access to push the docker image to staging, the steps will be outlined here.

The endpoint-app-team repository should be tagged with the same version number that is being released to the package-storage repo and
the contents of <./package/endpoint> and the package-storage repo should be equivalent.

The Makefile can be used to automate this process.

First install `hub` tool via `brew install hub`

Run `make release-package`

The `release-package` target will create a PR to package-storage. The `hub` tool will prompt for your github credentials and generate an
Oauth token. Once the token is created it will likely fail because SSO needs to be enabled. Browse to your github account and enable SSO on the token and
rerun the `make release-package` command.

The package version will be bumped once the command succeeds. Lastly, PR the version bump changes to this repo.

## Updating the schema

After making the necessary changes to the mapping, generate the new [schema files](schema/v1) as outlined in this [README](scripts/event_schema_generator/README.md)

Lastly, PR the changes to this repo.
