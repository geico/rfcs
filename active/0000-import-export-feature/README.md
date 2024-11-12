- Start Date: 2024-11-12
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Implementation PR(s): 

# Import/Export Feature

## Summary

This feature will add the ability to both export datasets to CSV files and import them back into DataHub from those CSV files, using the UI. The code for this feature is already complete, and in use internally at GEICO.

## Motivation

This feature was originally developed for GEICO's internal usage. It was intended to mimic the import/export functionality present in Collibra. The DataHub team has requested we contribute this feature back to their open source repository.

## Requirements

This feature as it is currently implemented is fairly limited, and only intended to support:
- Export to CSV of individual datasets.
- Export to CSV of containers of datasets.
- Import from CSV of previously exported data.

## Non-Requirements

This feature is not intended to add a REST API for import/export like that of Collibra. It is only intended for use through the UI.

## Detailed design

This feature will add three new options to the existing `SearchExtendedMenu` dropdown. One to export containers of datasets, one to export individual datasets, and one to import previously exported data into DataHub. The export options create CSV files from data existing in DataHub, while the import option adds new data to DataHub from CSV files. 

Below is a list of the column names used in the CSV files for this feature. Each row describes an individual dataset or dataset field. Presently, the `subresource` column is only used for dataset fields, while the `owners` and `domain` columns are only used for datasets. The `resource` column is used to store the dataset URN for both. The `subresource` column contains the names of dataset fields. The `glossary_terms`, `tags`, and `ownership_type` fields are presently unused. 

``` csv
resource,asset_type,subresource,glossary_terms,tags,owners,ownership_type,description,domain
```

### Export

Within `SearchExtendedMenu`, the container-level export option is only available when a container is being viewed. At all other times, it is grayed out and cannot be pressed. This is done using a react effect, which greys out the button unless the URL of the current page contains the word "container".

When either export option is selected, it opens a modal which prompts the user to enter the name of the CSV file to be created. For dataset-level export, the user is also prompted to enter the data source, database, schema, and table name of the dataset to be exported. Notably, these fields are specific to GEICO's use case and may need to be altered. This is what the fields refer to:
- Data source: The name of the data platform containing the dataset
- Database: A container representing a database within the data source
- Schema: A container representing a schema within the source database
- Table name: The name of the dataset

Upon entry, the following steps occur:

1. The modal is made invisible, but continues executing code for the export process. A notification is created to inform the user that the export process is ongoing.
2. The URN of the dataset or container is determined, by either:
    - Pulling from [`EntityContext`](https://github.com/datahub-project/datahub/blob/master/datahub-web-react/src/app/entity/shared/EntityContext.ts) in the case of container-level export.
    - Manually constructing the URN from data entered into the modal in the case of dataset-level export.
3. The modal uses the URN as input into GraphQL queries, which are used to fetch the metadata for the datasets to be exported.
    - Container-level export will first execute a GraphQL query to determine how many datasets are present in the container.
    - Additionally, container-level export will only fetch 50 datasets per GraphQL execution. If more than 50 datasets are present in the container, this query will be executed multiple times, with each execution producing and downloading separate CSV files.
4. The metadata returned from the GraphQL query is transformed into a CSV-compatible JSON object using a shared function, `convertToCSVRows`. Each row in this JSON object contains the columns described in the prior section.
5. The existing `downloadRowsAsCsv` function in [`csvUtils`](https://github.com/datahub-project/datahub/blob/master/datahub-web-react/src/app/search/utils/csvUtils.ts) is used to create the download.

#### GraphQl queries

These GraphQL queries are used for container-level export and dataset-level export, respectively:
``` graphql
query getDatasetByUrn($urn: String!, $start: Int!, $count: Int!) {
    search(input: { type: DATASET, query: $urn, start: $start, count: $count }) {
        start
        count
        total
        searchResults {
            entity {
                ... on Dataset {
                    urn
                    type
                    name
                    platform {
                        urn
                    }
                    domain {
                        associatedUrn
                        domain {
                            urn
                            type
                        }
                    }
                    properties {
                        name
                        description
                    }

                    editableProperties {
                        description
                    }

                    ownership {
                        owners {
                            associatedUrn
                        }
                    }
                    tags {
                        tags {
                            associatedUrn
                        }
                    }
                    glossaryTerms {
                        terms {
                            associatedUrn
                        }
                    }
                    editableSchemaMetadata {
                        editableSchemaFieldInfo {
                            description
                            fieldPath
                            tags {
                                tags {
                                    associatedUrn
                                }
                            }
                        }
                    }
                    schemaMetadata {
                        name
                        fields {
                            description
                            type
                            fieldPath
                            nativeDataType
                            tags {
                                tags {
                                    tag {
                                        name
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

query getTable($urn: String!, $start: Int!, $count: Int!) {
    search(
        input: {
            type: DATASET
            query: "*"
            start: $start
            count: $count
            orFilters: { and: [{ field: "urn", values: [$urn], condition: EQUAL }] }
        }
    ) {
        start
        count
        total
        searchResults {
            entity {
                ... on Dataset {
                    urn
                    type
                    name
                    platform {
                        urn
                    }
                    domain {
                        associatedUrn
                        domain {
                            urn
                            type
                        }
                    }
                    properties {
                        name
                        description
                    }

                    editableProperties {
                        description
                    }

                    ownership {
                        owners {
                            associatedUrn
                        }
                    }
                    tags {
                        tags {
                            associatedUrn
                        }
                    }
                    glossaryTerms {
                        terms {
                            associatedUrn
                        }
                    }
                    editableSchemaMetadata {
                        editableSchemaFieldInfo {
                            description
                            fieldPath
                            tags {
                                tags {
                                    associatedUrn
                                }
                            }
                        }
                    }
                    schemaMetadata {
                        name
                        fields {
                            description
                            type
                            fieldPath
                            nativeDataType
                            tags {
                                tags {
                                    tag {
                                        name
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

### Import

In the case of import, the button first opens a prompt to upload a file, using the following snippet of code.

``` jsx
<input id="file" type="file" onChange={changeHandler} style={{ opacity: 0 }} />
```

The `papaparse` library is used to parse the CSV file and iterate over each row present within it. The data is then fed into these GraphQL mutations to create datasets and dataset fields, respectively:

``` graphql
mutation updateDataset($urn: String!, $input: DatasetUpdateInput!) {
    updateDataset(urn: $urn, input: $input) {
        urn
    }
}

mutation updateDescription($input: DescriptionUpdateInput!) {
    updateDescription(input: $input)
}
```

Presently, only the `resource` and `description` CSV fields are used in the import of datasets, while only the `resource`, `description`, and `subresource` CSV fields are used in the import of dataset fields.

## How we teach this

> What names and terminology work best for these concepts and why? How is this idea best presented? As a continuation
> of existing DataHub patterns, or as a wholly new one?

> What audience or audiences would be impacted by this change? Just DataHub backend developers? Frontend developers?
> Users of the DataHub application itself?

> Would the acceptance of this proposal mean the DataHub guides must be re-organized or altered? Does it change how
> DataHub is taught to new users at any level?

> How should this feature be introduced and taught to existing audiences?

A user guide should be written on how to use the feature. In particular, we will need to highlight:
- Under what circumstances the container-level export button becomes available.
- How to fill out the form presented in the dataset-level export modal.

This feature would be best presented as a wholly new pattern. Though it is presently possible to download search results as CSV, the format of the resulting CSV files differs significantly from that of the import/export feature. As such, the files cannot be used as import input. 

## Drawbacks and Alternatives

As mentioned before, this feature is only intended for use within the UI. As the code has currently been written, it would not be possible to extend the import and export functionality to a different API (i.e., REST), as all the code is written in React. 

A possible alternative to this would be to move the code that performs the CSV data mutations and GraphQL querying to the Metadata Service or the Frontend Server, accessible from throughout the DataHub stack through a REST API or similar. This does not come without drawbacks, however. Namely, we would need to re-write the existing code entirely, and we'd be introducing additional complexity through the API endpoints. It's also notable that doing so is outside of the scope of GEICO's intent with this RFC, as we are simply hoping to share what we already have.

It's also notable that because the format of the CSV files is so different from those produced by the existing functionality of downloading search results, existing CSV files cannot be used to import datasets. This may cause confuion among users, and may be worth remediating.

It's also notable that a extension for DataHub does exist which adds very similar functionality ([link](https://datahubproject.io/docs/0.13.1/generated/ingestion/sources/csv/)). This has not been investigated in detail, but if this is a duplicate feature, it may not be worth integrating into DataHub.

## Rollout / Adoption Strategy

This feature does not change or break any existing functionality, and therefore no specific migration strategy is required.

## Future Work

As mentioned above, the `glossary_terms`, `tags`, and `ownership_type` CSV columns are presently unused. It would be fairly simple to add the functionality to fill those columns in, however, as we are already including the necessary fields in the search results of our GraphQL query.

## Unresolved questions

It's notable that the dataset-level export component of this feature was designed specifically for GEICO's use case, as it requires filling out a form that assumes a specific count of containers is in use for all datasets. This is unlikely to always be the case, and as such, this component will likely need to be refactored to be more flexible. We will need to determine what shape the component should take before performing this refactoring.