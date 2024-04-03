# Detection Engine

Detection engine acts as a datapipeline that can automate the movement and transformation of data. Using this one can define workflows, schedule tasks, run detections and publish the detection results to database

## Basic Structure


1. The module reads the configuration from database and loads it into memory.
2. The module then reads the data from the source and applies the transformations as defined in the configuration.
3. Based on the detection results the module then publishes the data to the detection collection/specific metadata collection

## Module Components


### 1. Detection Manager

Detection Manager, manages the lifecycle of different detections. It stores metadata about each detection, the data required by them, basic filtering condition etc. The primary responsibility of this class include

1. Maintaining the list of detection header, data required by them etc
2. Initializing the detection header and providing them with respective data
3. Managing the lifecycle of the detection header.

Detection managers can maintain multiple detection headers and runs them sequentially. Grouping multiple detection headers into a single detection manager can be useful if the subscribed headers make use overlapping data.

The basic schema structure of detection manager is as below:

```json
{
    'type': 'detection_manager',
    'collection_config': {
        <collection_name>: {
            columns: list of columns,
            last_id: last_id selected,
            time_diff: None | <num_timediff>,
            time_diff_unit: None(days) | 'W' | 'D' | 'H' | 'm' | 's'
            time_start: timestart_unit,
            time_end: timeend_unit,
            time_start_relative: None | <num_timestart_relative>,
            time_start_relative_unit: None(days) | 'W' | 'D' | 'H' | 'm' | 's',
            time_end_relative: None | <num_time_end_relative>,
            time_end_relative_unit: None(days) | 'W' | 'D' | 'H' | 'm' | 's',        
            conditions: [
                [
                    # and condition
                    {
                        key: value, 'match_operation': == | eq | > | gt | < | lt | <= | lte | >= | gte | ne | != | <>
                    },
                    {
                        key: value,
                        'match_operation': == | eq | > |gt | < | lt | <= | lte | >= | gte | ne | != | <>
                    },
                    {
                        key: value, 
                        'match_operation': == | eq | > | gt | < | lt | <= | lte | >= | gte | ne | != | <>
                    },
                ], 
                [
                    ...filter_condition_2
                ]
            ],
            is_multitenant: None | True | true | 1, # default is true
            column_regex: None | True | true | 1 | # default is false
            column_regex_orientation: start | end | any # default any

        },
        ...
    },
    detection_header': {
        <header_name>: {
            'type': 'header_type',
            'collection_name': 'name of the collection'
        },
        <header_name2>: {
            'type': 'header_type',
            'collection_name': 'name of the collection'
        }
    },
    default_selection: [#option list of columns to be projected by default]
}
```

Schema description for the same is as below

| Field | Description |
| ----  | ----------- |
| type  | Class type, always set to detection_manager |
| collection_config | A Hashmap of collections, which contains the list of collections and their data configuration required. The data is generalized across to all the header fields subscribed to it. |
| collection_config.collection_name | Name of the collection |
| collection_config.columns | List of columns to be selected from the collection |
| collection_config.last_id | `_id` after which the collection data needs to be selected |
| collection_config.time_diff | Time difference from the current time, after which the data needs to be selected |
| collection_config.time_diff_unit | Unit of the time difference, can be `W` for week, `D` for day, `H` for hour, `m` for minute, `s` for second |
| collection_config.time_start | Allows data to be selected from a specific period. It defines the start time of the period |
| collection_config.time_end | Allows data to be selected from a specific period. It defines the end time of the period, defaults to current datetime |
| collection_config.time_start_relative | Allows data to be selected from a specific period. It defines the start time of the period relative to the current time |
| collection_config.time_start_relative_unit | Unit of the time difference, can be `W` for week, `D` for day, `H` for hour, `m` for minute, `s` for second |
| collection_config.time_end_relative | Allows data to be selected from a specific period. It is only valid if `time_start_relative` is provided and `time_end_relative` is greater than the absolute value of `time_start_relative`. If not provided it defaults to current time and the expression would work simlar to `time_diff` |
|`time_end_relative_unit`| Unit of the time difference, can be `W` for week, `D` for day, `H` for hour, `m` for minute, `s` for second |
| collection_config.conditions | List of dictionary that needs to be applied aross the data. List here represents the `and` condition to be applied. Dictionary defines the `or` condition which can be further used for filtering. Each condition contains the map of operation to be performed where `map_key` represents the column name while `map_value` represents the value. Operation to be performed is indicated by `match_operation` field. Where the operation would be any one of the sets [`==`, `eq`] corresponds to `equals_to`, [`>`,`gt`] corresponds to greater than, [`<`,`lt`] corresponds to less than. [`<`,`lt`, `lte`] corresponds to less than or equal to, [`>`,`gte`] corresponds to greater than or equal to. [`!=`,`ne`, `<>`] corresponds to not equal to |
| detection_header | A hashmap of detection headers, which contains the list of detection headers and their configuration|
| detection_header.type | Defines the header `type` either `metadata` or `detection` |
| detection_header.collection | Collection where the results of the detections are stored |
| default_selection | List of columns to be selected by default, irrespective to whether the header has provided it or not.|

### 2. Detection Header

Detection Header corresponds to one single unit of detection that needs to be carried out. It contains the metadata of the detection, and contains the logic to perform sql like joins for two or more detections/collections. The primary responsibility of this class is to prepare the data required by the `detection` class and to send the data to `message queue`once the data is passed through all subscribed detections.

Following is the schema structure of the detection

```json
<header_name>: {
        # list of detections
        detections: {
                <detection_name_type1>: {
                    detection_type: "simple"
                    collection_name: <collection>,
                    columns: [list of columns to be selected],
                    conditions:conditions,
                    time_diff: None | unit,
                    time_diff_unit: None | W | D  | h | m | s,
                    last_id: None,
                    time_start: <start_time>,
                    time_end:  <end_time>,
                    time_start_relative: None | <int_diff>,
                    time_start_relative_unit: None | W | D | h | m | s,
                    time_end_relative: None | <int_diff>,
                    time_end_relative_unit: None | W | D | h | m | s,
                    update_last_id: None | 1 | -1,
                    is_multitenant: None | 1 | -1,
                    column_regex: None | 1 | -1,
                    column_regex_orientation: start | end | any,

                },
                <detection_name_type2>:{
                    detection_type: "complex"
                    collection_name: {"detection_name1": [list of columns to join on], 
                        "detection_name2": [list of colunns] 
                    },
                    drop_duplicates: "drop" | "left" | "right" | "preserve", // default preserve 
                    join_direction: 0 | 1 # 0: horizontal (default) 1: vertical, 
                    join_type: "inner"| "left" | "outer" | "right",
                },
                <detection_name_type3>: {
                    detection_type: "complex"
                    collection_name: {
                        "detection_name1"|"collection_name1": {}, # self join 
                    },
                },
                <detection_name_type4>: {
                    detection_type: "complex-select",
                    collection_name: {
                        <collection_name>: [columns to join on],
                        <detection_name>: [columns to join on],
                        match_condition: [
                            {
                                collection_column_name: condition, <allowed condition include: eq, ne, gt, lt, gte, lte>

                            }

                        ]
                    },
                    columns: [columns to project from collection name],
                    time_diff: None | <int_diff>,
                    time_diff_unit: None | "M" | W | D | h | m | s,
                    last_id: None, | ObjectId (collection name)
                    time_start: <start time>,
                    time_end: end_time,
                    is_multitenant: None | 1 | -1,
                    time_start_relative: None | <int_diff>,
                    time_start_relative_unit: None | "M" | W | D | h | m | s,
                    time_end_relative: None | <int_diff>,
                    time_end_relative_unit: None | "M" | W | D | h | m | s,
                    batch_size: None | <int_batch_size>, # defaults to 1000
                    join_type: "inner" | "left" | "outer" | "right", # defaults to inner
                    join_direction: 0 | 1 # 0: horizontal (default) 1: vertical
                },
        detection_tags: {
            detection_tag1: [array of column names],
            detection_tag2: [array of column names],
            ..
            detection_tag3: [array of column names],
        },
        # passes calculated fields to be displayed
        # by UI components
        fields_to_dashboard: [column1, column2... column(n)] 
        status: "draft"|"published",
        # from detection header
        header_type: metadata | detection,
        collection_name: <name_of_metadata_collection>
        remediations: {
            <res_column_name>: remediation_template_name
            conditions: {
                <condition_column_name> : {
                    column_name: column_value
                    match_condition: eq | ne | gt | lt ... 
                }
            }
        }
    }
```

| Field | Description |
| ----  | ----------- |
| header_name | Name of the detection header |
| detection_tags | A hashmap of detection tags and the columns that needs to be displayed in the dashboard |
| fields_to_dashboard | List of columns that needs to be displayed in the dashboard |
| status | Status of the detection header, can be `draft` or `published` |
| header_type | Type of the header, can be `metadata` or `detection` |
| collection_name | Name of the collection where the detection results are stored |
| detections | Contains the metadata information for the list of detections subscribed to the header |
| detections.detection_name | Name of the detection |
| detections.type | Type of the detection. Currently following 3 formats of detects are supported 1. `simple`: most atomic detection has no dependency on other detection or rules 2. `complex`: Allows joining the data of one or more rules/collection 3. `complex-select`: Allows joining the data of an existing detection and lazy loading the collection data from an other collection, in this scenario the existing rule acts as a filtering condition for selecting data for the lazy loading |
| `detections.detection_name.type == 'simple'` | *following fields are only defined for the detection type as simple*  |
| detections.detection_name.collection_name | Name of the collection |
| detections.detection_name.columns | List of columns to be selected from the collection |
| detections.detection_name.conditions | Additional filtering condition |
| detections.detection_name.time_diff | Allows time based filtering with delta being the `time_diff` field |
| detections.detection_name.time_diff_unit | Unit of the time difference, can be `W` for week, `D` for day, `H` for hour, `m` for minute, `s` for second |
| detections.detection_name.update_last_id | `_id` should it be updated after rule is executed. `1` to update `-1`  not to update. Default value is `1` if None is provided |
| detections.detection_name.last_id | `_id` after which the collection data needs to be selected |
| detections.detection_name.time_start | Allows data to be selected from a specific period. It defines the start time of the period |
| detections.detection_name.time_end | Allows data to be selected from a specific period. It defines the end time of the period, defaults to current datetime |
| detections.detection_name.time_start_relative | Allows data to be selected from a specific period. It defines the start time of the period relative to the current time |
| detections.detection_name.time_start_relative_unit | Unit of the time difference, can be `W` for week, `D` for day, `H` for hour, `m` for minute, `s` for second |
| detections.detection_name.time_end_relative | Allows data to be selected from a specific period. It is only valid if `time_start_relative` is provided and `time_end_relative` is greater than the absolute value of `time_start_relative`. If not provided it defaults to current time and the expression would work simlar to `time_diff` |
| detections.detection_name.time_end_relative_unit | Unit of the time difference, can be `W` for week, `D` for day, `H` for hour, `m` for minute, `s` for second |
| detections.detection_name.is_multitenant | If set to -1 will disable multitenancy while selecting records from the collection |
| detections.detection_name.column_regex | If set to true, columns are assumed to be a regex expression and filters all column names which matches the regex fields. Selection strategy is defined by `column_regex_orientation`  |
| detections.detection_name.column_regex_orientation | Defines the orientation of the regex expression. Can be `start` for matching the start of the column name, `end` for matching the end of the column name, `any` for matching any part of the column name |
| `detections.detection_name.type == 'complex'` | *following fields are only defined for the detection type as complex*  |
| detections.detection_name.collection_name | A hashmap of collection/detection name and the columns to be joined on. If the hashmap has only one key-value pair then it is considered as the configuration for `self-join`. The data for the collection is passed as is. Else the joining condition is defined using `join_type` |
| detections.detection_name.join_direction | Defines the direction of the join. `0`: defaults to horizontal join, `1`: corresponding to vertical join |
| detections.detection_name.join_type | Defines the join_type. It can either be `inner`: corresponds the sql inner join, `left`: corresponds to sql left join, `right`: corresponds to sql right join. `outer`: corresponds to sql outer join; If join direction is vertical then only join type supported are `inner`: Intersection `outer`: Union, `left`: keeps the left side dataframe intact, `right`: keeps the right side of the dataframe intact |
| detections.detection_name.drop_duplicates | Defines the strategy to handle duplicates. `drop`: drops the duplicates keeping `_id` as the unique, `left`: keeps the left side intact keeping the columns for the left side intact, `right`: keeps the right side intact, `preserve`: preserves the duplicates keeping both left and right side |
| `detections.detection_name.type == 'complex-select'` | *following fields are only defined for the detection type as complex-select*  |
| detections.detection_name.collection_name | A hashmap of collection/detection name and the columns to be joined on. The hashmap needs to have 2 keys one of which corresponds to the existing detection_name while the other corresponds to the collection name  and/or a match conditions|
| detections.detection_name.collection_name.match_condition | Specifies dynamic mapping of the column name for the collection name. The match condition contains columns corresponding `colletion_name`; For the detection data passed the query is build at the run time based on the config provided to the match_condition. The possible match conditions include `lt` , `lte`, `<`,`<=`,`gt`,`>`, `gte`, `>=`, `ne` `!=`, `<>` and `eq`, `==`. It defaults to 'eq'. *Note* the operation is very expensive and should be used only when required. For further reference check [appendix](#conditional-joining-in-complex-select) |
| detections.detection_name.column_name | Name of the columns that needs to be projected |
| detections.detection_name.time_diff | Allows additional time based filtering, defines the difference from the current time |
| detections.detection_name.time_diff_unit | Defines the unit of time_diff `W`: corresponds to week, `D`: Corresponds to day, `h`: corresponds to hour,`m`: correponds to minutes, `s`: Corresponds to seconds  |
| detections.detection_name.last_id | Defines the `_id` after which data needs to be selected |
| detections.detection_name.time_start | Allows data to be selected from a specific period. It defines the start time of the period |
| detections.detection_name.time_end | Allows data to be selected from a specific period. It defines the end time of the period, defaults to current datetime |
| detection.detection_name.is_multitenant | If set to -1 will disable multitenancy while selecting records from the collection |
| detections.detection_name.time_start_relative | Allows data to be selected from a specific period. It defines the start time of the period relative to the current time |
| detections.detection_name.time_end_relative | Allows data to be selected from a specific period. It defines the end time of the period relative to the current time |
| detections.detection_name.time_start_relative_unit | Unit of the time difference, can be `W` for week, `D` for day, `H` for hour, `m` for minute, `s` for second |
| detections.detection_name.time_end_relative_unit | Unit of the time difference, can be `W` for week, `D` for day, `H` for hour, `m` for minute, `s` for second |
| detections.detection_name.is_multitenant |  If set to -1 will disable multitenancy while selecting records from the collection |
| detections.detection_name.join_direction | Specifies the join direction `0`: horizontal (default), `1`: vertical |
| detections.detection_name.join_type | Specifies the join type. For join direction horizontal allowed join types include `inner`, `outer`, `left` and `right`. For join direction vertical allowed join types include `inner`: intersection, `outer`: union, `left`: keeps the left column intact, `right`: keeps the right columns intact |
| remediations | A hashmap of remediation template name and the column name to which it needs to be applied  Seperate remediation templates can be applied to different columns. The remediation template is applied to the column only if the condition is satisfied. The condition is defined by the `conditions` field. For details [refer](#remediation-template) |
|

## 3. Detections

Detections are the most atomic unit of the detection engine. The class is responsible for granular matching and comparisons. The class will have the following schema structure

```json
    {
        <detection_name>: {
            <method> : [
                    { # pipe method1
                    'columns': [list of columns],
                    'function_names': {
                        'function_name': {
                                    function_param1: function_param1_value,
                                    function_param2: function_param2_value,
                                    function_param3: function_param3_value,
                                }
                            }
                    },
                    ...
                    { # pipe methodn
                    'columns': [list of columns],
                    'function_names': {
                        'function_name': {
                                    function_param1: function_param1_value,
                                    function_param2: function_param2_value,
                                    function_param3: function_param3_value,
                                }
                            }
                    }
                ]
            }
    }
```

Details of the schema structure is as follows

| Field | Description |
| ----  | ----------- |
| detection_name | Name of the detection |
| method | Name of the method to be applied |
| columns | List of columns to be selected from the collection on which operation is perfomed|
| function_names | A hashmap of auxillary function name and the parameters to be passed to the function |

Following are the list of core functions supported by the module

<table>
<tr>
<th> Function Name </th><th> Description</th><th> Grammar </th>
</tr>
<tr>
<td> in </td><td> Checks if the column contains values in specified data </td><td>

```json
in: [
            {
                <column_name1>: ['value1','value2','value3'..'value_n'],
                function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
                }
            },
        ]
```

</td>
</tr>
<td> not_in </td><td>  Checks if the column does not contain values in specified list </td> <td>

```json
not_in: [
            {
                <column_name1>: ['value1','value2','value3'..'value_n'],
                function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
                }
            },
        ]
```

</td>
</tr>
<tr>
<td> substring </td><td>Checks if the column value matches a substring</td><td>

```json
substr: [
            {
                column_name: match_condition # add support for list? 
                function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
                }
            }
        ]
```

</td>
</tr>
<tr>
<td>not_substring</td><td>Checks if the column value does not matches a substring</td><td>

```json

not_substring: [
                {
                    column_name: match_condition # add support for list? 
                    function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
                    }
                }
            ]
```

</td>
</tr>
<tr>
<td> equal , eq, = </td><td>Checks if the column value matches a specified input</td><td>

```json
eq: [
        {
            # config_key : config_value
            "column_name1": "value1"
            function_names: {
                "auxilliary_function_name":{
                        "function_arg1":"function_arg1_value",
                        "function_arg2":"function_arg2_value",
                        ...
                    }
            },
        }
        {
                "column_name2": "value1"
                function_names: {
                        "auxilliary_function_name":{
                        "function_arg1":"function_arg1_value",
                        "function_arg2":"function_arg2_value",
                        ...
                        }
                },
        }
    ]
```

</td>
</tr>
<tr>
<td> ne, !=, <> </td><td>Checks if the column value does not matches the specific value</td><td>

```json
ne: [
        {
        column_name: column_value,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
        }
]

```

</td>
</tr>
<tr>
<td> gt, >, greater_than </td><td>Checks if the column value is greater than the specified value</td><td>

```json
gt: [
        {
        column_name: column_value # add support for list? 
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    ]
```

</td>
</tr>
<tr>
<td> gte, >=, greater_than_equal </td><td>Checks if the column value is greater or equal to the specified value</td><td>

```json

gte: [
        {
            column_name: column_value # add support for list? 
            function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
            }
        }
    ]
```

</td>
</tr>
<tr>
<td> lt, <, less_than </td><td>Checks if the column value is less than the specified value</td><td>

```json
lt: [
        {
            column_name: column_value # add support for list? 
            function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
            }
        }
    ]
```

</td>
</tr>
<td> lte, <=, less_than_equal </td><td>Checks if the column value is less or equal to the specified value</td><td>

```json
lte: [
        {
        column_name: column_value # add support for list? 
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    ]
```

</td>
<tr>
<td> or, | </td><td>Checks if the column value matches at least one of the conditions</td><td>

```json
or: {
    condition1: condition1,
    condition2: condition2
}
```

</td>
</tr>
<tr>
<td>and, & </td></td><td>Check of the column matches all the conditions</td><td>

```json
and: {
    condition1: condition1,
    condition2: condition2
}
```

</td>
</tr>
<tr>
<td>column_match</td></td>checks if column matches (one to one) with an other column in dataframe<td></td><td>

```json
col_match: [
    {
        column1 : column2,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    },
    {
    column2 : column3,
    function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
    }
    }
]
```

</td>
</tr>
<tr>
<td> not_column_match </td><td>checks if column does not match (one to one) with an other column in dataframe</td><td>

```json
not_col_match: [
    {
        column1 : column2,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    },
    {
        column2 : column3,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    }
]
```

</td>
</tr>
<tr>
<td> column_lt </td><td>checks if the column left column is less than the right column</td><td>

```json
col_lt: [
    {
        column1 : column2,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    },
    {
        column2 : column3,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    }
]
```

</td>
<tr>
<td> column_lte </td><td>checks if the column left column is greater than the right column</td><td>

```json
col_gt: [
    {
        column1 : column2,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    },
    {
        column2 : column3,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    }
]
```

</td>
</tr>
<tr>
<td>column_gt </td><td></td><td>

```json
col_gt: [
    {
        column1 : column2,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    },
    {
        column2 : column3,
       function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
       }
    }
    ]
```

</td>
</tr>
<tr>
<td>column_gte</td><td>checks if the column left column is greater than or equal to the right column</td><td>

```json
col_gte: [
    {
        column1 : column2,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    },
    {
        column2 : column3,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    ]

```

</td>
</tr>
<td>column_eq</td><td>checks if column matches (one to one) with an other column in dataframe</td><td>

```json
col_match: [
    {
        column1 : column2,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    },
    {
        column2 : column3,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    }
    ]

```

</td>
</tr>
<tr>
<td> column_ne</td><td>checks if column does not match (one to one) with an other column in dataframe</td><td>

```json
not_col_match: [
    {
        column1 : column2,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    },
    {
        column2 : column3,
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    }
    ]

```

</td>
</tr>
<tr>
<td> project </td><td>Allows selecting of individual columns/columns and apply specific functionality/functions on top of it </td><td>

```json
'project': [
        {
        columns: [column_name1, column_name2,column_name3 .. ],
        function_names: {
                        "auxilliary_function_name":{
                            "function_arg1":"function_arg1_value",
                            "function_arg2":"function_arg2_value",
                            ...
                            }
        }
    }
}
]
```
</td>
</tr>
<tr>
<td> detections </td><td>Evaluates the expression (similar to "and" condition) and returns a true/false value</td><td>

```json
'detect': [
    {
        dest_cols: Optional // defaults to detections
            operation: {
                op: operands
            }
    }
]
```

</td>
</tr>
<tr>
<td> time_gte </td><td>Time based operation, it compares the specified column values to be greater than or equal to the current value. The columns provided would be converted into timestamp representation for this conversion</td><td>

```json
time_gte: [
    { column1: value1, function_names: { 'timestamp'}},
]
```

</td>
</tr>
<tr>
<td> time_gt </td><td>Time based operation, it compares the specified column values to be greater than the current value. The columns provided would be converted into timestamp representation for this conversion</td><td>

```json
time_gt: [
    { column1: value1, function_names: { 'timestamp'}},
]
```

</td>
</tr>
<tr>
<td> time_lt </td><td>Time based operation, it compares the specified column values to be less than  the current value. The columns provided would be converted into timestamp representation for this conversion 
</td><td>

```json
time_lt: [
    { column1: value1, function_names: { 'timestamp'}},
]
```

</td>
</tr>
<tr>
<td> time_lte </td><td>Time based operation, it compares the specified column values to be less than or equal to the current value. The columns provided would be converted into timestamp representation for this conversion
</td><td>

```json
time_lte: [
    { column1: value1, function_names: { 'timestamp'}},
]
```

</td>
</tr>
<tr>
<td> time_eq </td><td>Time based operation, it compares the specified column values to be equal to the current value. The columns provided would be converted into timestamp representation for this conversion
</td><td>

```json
time_eq: [
    { column1: value1, function_names: { 'timestamp'}},
]
```

</td>
</tr>
<tr>
<td> time_ne </td><td>Time based operation, it compares the specified column values to be not equal of the current value. The columns provided would be converted into timestamp representation for this conversion
</td><td>

```json
time_ne: [
    { column1: value1, function_names: { 'timestamp'}},
]
```

</td>
</tr>
<tr>
<td> isna </td><td>For a specified column/columns the function filters NA or None values
</td><td>

```json
isna: [
    {column1: {
        function_names: {
                function_name: {argument_map}
        }
    }},
    column2,
    column3,
    {
        column_n: {
            function_names: {
                function_name: {argument_map}
            }
        }
    }
]
isna: {
    column1: {
        function_names: {
            function_name: {argument_map}
        }
    }
}
isna: column1
```

</td>
</tr>
<tr>
<td> isnotna </td><td>For specified column/columns the function filters out the Non NA values
</td><td>

```json
isnotna: [
    {column1: {
        function_names: {
                function_name: {argument_map}
        }
    }},
    column2,
    column3,
    {
        column_n: {
            function_names: {
                function_name: {argument_map}
            }
        }
    },
]
isnotna: {
    column1: {
        function_names: {
            function_name: {argument_map}
        }
    }
}
isnotna: column1
```

</td>
</tr>
<tr>
<td> column_exists </td><td> Checks if the column name specified in the configuration exists in the data passed. Returns the data if column exists, else returns None
</td><td>

```json
column_exists: [..list of column names]

column_exists: column_name
```

</td>
</tr>
<tr>
<td> not_column_exists </td><td> Checks if the column name specified in the configuration does not exists in the data passed. Returns the data if column does not exists, else returns None
</td><td>

```json
not_column_exists: [..list of column names]

not_column_exists: column_name
```

</td>
</tr>
</table>

### Auxillary Functions

Following are the list of auxillary functions supported

| Function Name | Family | Description | parameters |
| ------------- | -------|  ----------- | ---------- |
| rank_order | aggregate | Returns data ordered by rank method for the dataframe [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.rank.html)| |
| data_range | aggregate | Gets the data range for numeric data grouped by categorical data [refer](https://pandas.pydata.org/docs/reference/api/pandas.date_range.html)| categorical_cols: columns with respect to which the range is calculated, value_cols: the column representing the numeric data, return_original: If the range_value is added back to the original dataframe |
|get_topn| aggregate| Splunk function: Finds the most common values for the fields in the list. Calculates the count and a percentage of the frequency the values occur in the event. If there is a by clause included then result is grouped by the `by-clause` [refer](https://docs.splunk.com/Documentation/Splunk/9.1.0/SearchReference/Top)| field_counts: Calculates the rank for the required field names, groupby_cols: columns to be grouped by, top_value: number of top entry values requested |
|transform_project| aggregate| Performs transformation function a grouped columns. It produces a dataframe with the same axis shape as self Returns the result of the transformed data [refer](https://pandas.pydata.org/docs/reference/api/pandas.core.groupby.DataFrameGroupBy.transform.html)|`groupby_col`: columns to group data on,`source_col`: columns on which to perform aggregation function,`groupby_operation`: the groupby operation to perform,`dest_cols`: columns where the data is stored|
|transform_filter| aggregate | Performs transformation function a grouped columns. It produces a dataframe with the same axis shape as self. Filters the dataframe columns based on the transformation performed. | `groupby_col`: columns to group data on;`source_col`: columns on which to perform aggregation function; `groupby_operation`: the groupby operation to perform; `dest_cols`: columns where the data is stored.|
|agg| aggregation | Defines the aggregation pipeline for the dataframe under consideration. Current detection pipeline supports the following [functions](#aggregate-functions). For details of working of an aggregate pipeline [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.aggregate.html) | Following parameters are supported `groupby_cols`: columns to group data on; `groupby_operation`: Map that defines the groupby operation `merge`: If true the aggregation results wll be merged back to the original dataframe  |
|value_counts| Returns a series/dataframe returning the count of unique values. The resulting object will be in descending order with most frequent value on top. [refer](https://pandas.pydata.org/docs/reference/api/pandas.Series.value_counts.html)| `groupby_cols`: columns to group data on; `raname`: provides a customize naming option for the the dataframe. `normalize`: If thrue then the object returned will contain the relative frequencies of the unique values. `sort`: Sort by frequencies `ascending`: Sort in ascending order; `drop_na`: Do not count the values which are `NaN`  |
|time_agg| aggregation | Time based aggregation. It allows aggregation on different column name parameters. Set of operations [supported](#time-based-aggregate-operations) | Needed/supported parameters include  `timestamp_col`: column which needs to be set as a timestamp representation/index. `groupby_col`: Additional column names used to perform grouping operation. `operation`: Operation Needed to be performed. Operations would be performed on individual column name . `rename`: Renames the column presented in aggregation results.|
|transaction| aggregation | [Splunk function](https://docs.splunk.com/Documentation/Splunk/9.1.0/SearchReference/Transaction). The transaction comman finds the transaction based on the events that meet the various constraints. The transaction are made up of the raw text of each member. The time and date fields of the earliest member. Additionally the dunction adds in two field 1. duration 2. eventcount. The values in the duration field shows the difference between the timestamps for the first and ladt events in the transaction. The values in the event count field show the events in the transaction. A transaction search is useful for a single observation of any type of event stretching over multiple logged events. e.g. set of events to firewall intrusion detection incident. | `timestamp_col`: column name on which the transaction is called. `groupby_col`: Optional columns on which the data is grouped. `maxspan`: Maximum length of time in seconds, minutesm hours or days; `maxevents`: maxumum events per transaction `maxpause`: maximum span between 2 events|
|isolation_forest| anomaly | Isolation forest Algorithm. Return the anomaly score of each same using IsolationForest Algorithm. The isolationforest 'isolates' observations by randomly selecting features and then randomly selecting a split value between the maximum and minimum values of the selected feature. Since recursive partitioning can be represented by a tree structure, the number of splitting required to isolate a sample is equivalent to the path length from the root node to the terminating node. The path length, average over a forest of such random trees, is a measure of normality and our decision function. Random partiioning produces noticeably shorter path for anomalies. When a forest of random trees collectively produce shorter path lengths for particular samples, they are higly likely to be anomalies [refer](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.IsolationForest.html)| `source_cols`: (optional) subset of columns on which anomaly detection is carried out, source_cols is empty the entire dataframe is considered for anomaly detection; `transformed_cols`: (optional) subset of columns where the transformed data is stored. `transformation_name`: if transformation is performed it specifies the name of the transformation to be performed; `save_model`: boolean value to indicate the whether to save model after training. `retrain`: Boolean field, if true the algo will pull an already trained model from mongodb. `detection_name`: Name of the detection, applicable for saving the model. `model_file_name`: Name of the file when saved in mongodb. `cust_id`: Optional customer id. `tenant_id`: Optional Tenant id `service_provider`: Service provider specificartion. `service_type`: Sub type of the specifciation. `return_score`: boolean field; if true returns it appends the the score given by the algorithm. `return_anomaly`: boolean field; of true it returns the label if its anomaly or not. `filter_results`: if true, it will return anomalous results only; `n_estimators`: The number of base estimators in the ensemble; `max_samples`: The number of samples to draw from data to train the base estimator; `contamination`: The amount of contamination of the data set, i.e. proportion of outliers in the data set. Used when fitting to define the threshold on score of the samples. if `auto` , the threshold is determined as in the original paper. if `float`, the contamination should be in range (0,0.5]. `max_features`: The number of features to draw from X to train each base estimator. `bootstrap`: If True, individual trees are fit on random subsets of the training data. `random_state`: int; controls the pseudo randomness of the selection of the features. `verbose`: Controls the verbosity of the tree building process. `warm_start`: bool , When set to True, reuse the solution of the previous call to fit and add more estimators to esemble, otherwise fit the whole new forest|
|robust_covariance| anomaly | It performs outlier detection for data distributed in a Guassian distributed dataset. One common way of performing outlier detection is to assume that the regular data come from a known distribution. It fits a robust covariance estimate to the data and thus fits an ellipse to the central data points, ignoring points outside the central mode. [refer](https://scikit-learn.org/stable/modules/generated/sklearn.covariance.EllipticEnvelope.html)| `source_cols`: (optional) subset of columns on which anomaly detection is carried out; source_cols is empty the entire dataframe is considered for anomaly detection. `transformed_cols`: (optional) subset of columns where the transformed data is stored. `transformation_name`: if transformation is performed it specifies the name of the transformation to be performed `tranformation_params`: dictionary, provides extra auxillary parameter for specific transformation. `save_model`: boolean value to indicate whether to save model after training. `retrain`: Boolean field, if true the algo will pull an already existing algorithm from mongodb. `detection_name`: Name of the detection name, applicable if needing to save the model.`model_file_name`: Unique file name for the saved model `cust_id`: Optional customer id. `tenant_id`: Optional tenant id. `service_provider`: service provider specification. `service_type`: sub type of the specification. `return_score`: boolean field; if true returns it appends the the score given by the algorithm. `return_anomaly`: boolean field; of true it returns the label if its anomaly or not. `filter_results`: if true; it will return anomalous results only .`assume_centered`: If True the support of robust location and covariance estimates is computed, and a covariance estimate is recomputed from it, without centering the data. `support_fraction`: The portion of points to be uncluded in the support of the raw MCD estimate. `contamination`: The amount of contamination of the data set. `random_state`: Determines the pseudo random generator for shuffling the data|
|one_class_svm| anomaly | Unsupervised outlier detection. Estimate the support of a high dimensional distribution. It is sensitive to outlier and thus does not perform well for outlier detection. The estimator is best suited for novelty detection when the training set is not contaminated by outliers.| `source_cols`: (optional) subset of columns on which anomaly detection is carried out.source_cols is empty the entire dataframe is considered for anomaly detection. `transformed_cols`: (optional) subset of columns where the transformed data is stored.`transformation_name`: if transformation is performed it specifies the name of the transformation to be performed. `tranformation_params`: dictionary, provides extra auxillary parameter for specific transformation.`save_model`: boolean value to indicate whether to save model after training. `retrain`: Boolean field, if true the algo will pull an already existing algorithm from mongodb. `detection_name`: Name of the detection name, applicable if needing to save the model.`model_file_name`: Unique file name for the saved model. `cust_id`: Optional customer id. `tenant_id`: Optional tenant id. `service_provider`: service provider specification. `service_type`: sub type of the specification. `return_anomaly`: boolean field; of true it returns the label if its anomaly or not. `filter_results`: if true; it will return anomalous results only. `kernel`: Specifies the type to be used in algorithm. If none is given 'rbf' will be used ,allowed values include linear, poly, rbf, sigmoid, precomputed.`degree`: Degree of the polynomial kernal, must be non negative. `gamma`: {'scale', 'auto'} or floatKernel corfficent for 'rbf', 'poly' nd sigmoid.`coef0`: Independent term in kernel function. `tol`: Tolerance for stopping criterion. `nu`: An upper bound on the fraction of training weeors and a lower bound of fraction of support vectors. Should be in the interval (0,1]; `shrinking`: Whether to use the shrinking heuristic.`cache_size`: Specify the size of the kernel cache (in MB). `max_iter`: Hard limit of iteration within solver, or -1 for no limit |
|one_class_svm_sgd| anomaly | it is an implementation of the one class SVM based on stochastic gradient descent (SGD). Combined with kernel approximation the solution of a kernelized. The main advantage of this method linearly with the number of samples [refer](https://scikit-learn.org/stable/modules/generated/sklearn.svm.OneClassSVM.html)| `source_cols`: (optional) subset of columns on which anomaly detection is carried out, source_cols is empty the entire dataframe is considered for anomaly detection. `transformed_cols`: (optional) subset of columns where the transformed data is stored. `transformation_name`: if transformation is performed it specifies the name of the transformation to be performed. `tranformation_params`: dictionary, provides extra auxillary parameter for specific transformation. `save_model`: boolean value to indicate whether to save model after training. `retrain`: Boolean field, if true the algo will pull an already existing algorithm from mongodb. `detection_name`: Name of the detection name, applicable if needing to save the model. `model_file_name`: Unique file name for the saved model. `cust_id`: Optional customer id ,`tenant_id`: Optional tenant id; `service_provider`: service provider specification. `service_type`: sub type of the specification. `return_score`: boolean field; if true returns it appends the the score given by the algorithm. `return_anomaly`: boolean field; of true it returns the label if its anomaly or not. `filter_results`: if true; it will return anomalous results only. `nystroem_kernel`: Kernel map to be approximated. `nystroem_gamma`: Gamma parameters for the rbf.`nystroem_coef0`: Zero coefficient for polynomial and sigmoid kernels. `nystroem_n_components`: Number of features to construct.`sgd_nu`: The nu parameter for One Class SVM: an upper bound on the fraction of training error. `sgd_fit_intercept`: Whether the intercept to be intercepted or not. `sgd_max_iter`: Maximum number of passess over training data.`sgd_tol`: The stopping creiteria `sgd_shuffle`: Whether or not the training data should be shuffled after each epoch. `sgd_learning_rate`: The learning rate scheduled to use fit. `constant`: eta= eta0. `optimal`: eta = 1.0 / (alpha * (t+t0)) where t0 is chosen by a heuristic approach. `invscaling` eta = eta0 / pow(t, power_t). `adaptive`: eta = eta0  as long as training keeps decreasing. `sgd_eta0`: float; the initial learning rate for 'constant', 'invscaling' or 'adaptive'. `sgd_power_t`: The exponent for inverse scaling learning rate (default 0.5). `sgd_warm_start`: When set to True, reuse the solution of the previous call to fit the initialization. `sgd_average`: When set to true, it computes the averaged SGD weights and stores the result in coef_ attribute |
|local_outlier_factor| anomaly | It is a mechanism to determin outlier in a moderately high dimensional datasets. It computes a score (called outlier factor) reflecting the degree of abnormalty of the observations. It measures the local density deviation of a given data point with respect to its neighbor. The idea is to detect the samples we have a substantially lower density than their neighnors. Algorithm works by forming the k-nearest neighbors. The LOF scores of an observation is equal to the ratio of average local density of its k neighbors and its own local density, a normal instance is expected to have a local density similar to that of its neighbors, while abnormal data are expected to have a smaller local density. The number k of neighbors considered greater than the minimum number of objects a cluster has to contain. So objects that can potentially be outliers. The strength f the LOF algorithm is that it takes both the local and global properties of the dataset into consideration. it can perfom well even in datasets where abnormal samples have different underlying densities.| `source_cols`: (optional) subset of columns on which anomaly detection is carried out.source_cols is empty the entire dataframe is considered for anomaly detection. `transformed_cols`: (optional) subset of columns where the transformed data is stored. `transformation_name`: if transformation is performed it specifies the name of the transformation to be performed . `tranformation_params`: dictionary, provides extra auxillary parameter for specific transformation. `save_model`: boolean value to indicate whether to save model after training. `retrain`: Boolean field, if true the algo will pull an already existing algorithm from mongodb. `detection_name`: Name of the detection name, applicable if needing to save the model.`model_file_name`: Unique file name for the saved model . `cust_id`: Optional customer id;`tenant_id`: Optional tenant id; `service_provider`: service provider specification;`service_type`: sub type of the specification;`return_score`: boolean field; if true returns it appends the the score given by the algorithm;`return_anomaly`: boolean field; of true it returns the label if its anomaly or not;`filter_results`: if true; it will return anomalous results only; `n_neighbors`: Number of neighbors to use;`algorithm`: Algorithm used to compute the nearest neighbor, allowed values are `ball_tree` , `kd_tree`, `brute`, `auto`. `leaf_size`: Leaf Size passed to  Ball tree or KDTree. This can affect the speed of the construction and query as well as the memory requried to store the tree. `metric`: Metric used for distance computation. Default is minkowski which results in standard euclidean distance valid metrics include `cityblock`,`cosine`, `euclidean`,  `haversine`,`l1`, `l2`,`manhattan`,`nan_euclidean`; `pairwise_distance:` Parameter for minikowski metric; `contamination:` The amount of contamination in the dataset |
|histogram| anomaly | Splunk function; Calculates the histogram based on frequency distribution; [refer](https://docs.splunk.com/Documentation/MLApp/5.4.0/User/SmartOutlierAssistant)| `source_cols`: columns on which histogram detection is carried out;`transformed_cols`: (Optional) Intermediate columns storing transformed data;`transformation_name`: Type of data transformation needed to be performed; `pthresh`: float; describes the minimum threshold for filtering the values; If not provided then the optimal value is calculated; `return_prob`: boolean to indicate whether to return probability; `filter_results`: If true the function returns all anomalous records.|
|iqr| anomaly | Function calculates the anomaly based on iqr scoring. The method is useful for continuous distribution. Formula used for outlier is `outlier = value>(q3+param*iqr) po value<(q1-param*iqr)`; where q3 is the 75th quantile and q1 is the 25th quantile. | `source_cols`: column under requiring iqr based anomaly detection; `transformation_name`: name of the transformation function that needs to be applied; `transformed_cols`: columns where transformation data is stored; `param`: multiplying factor.`min_quantile`: minimum quantile factor;`max_quantile`: maximum quantile factor|
|zscore| anomaly | Anomaly detection based on [zscore](https://en.wikipedia.org/wiki/Standard_score) | `column`: column on which the filtering is perform; `percentage_value`: percentage value determining the threshold|
|add, +| arithematic | Get addition of dataframe using elementwise binary operation add. The operation is equivalent to dataframe + other [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.add.html)| `source_cols`: source columns on which the dataframe operation needs to be performed;`dest_cols`: place where to store results;`operands`: scalar, sequence, series, dict value; `fill_value`: if specified then it fills the na value by this default value.|
|sub, -| arithematic | Get substract of dataframe using elementwise binary operation sub. The operation is equivalent to dataframe - other [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.sub.html)| `source_cols`: source columns on which the dataframe operation needs to be performed;`dest_cols`: place where to store results; `operands`: scalar, sequence, series, dict value; `fill_value`: if specified then it fills the na value by this default value;`right`: boolean to indicate whether to perform right substraction|
|div, /| arithematic | Get division of dataframe using elementwise binary operation /. The operation is equivalent to dataframe / other [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.div.html)| `source_cols`: source columns on which the dataframe operation needs to be performed;`dest_cols`: place where to store results;`operands`: scalar, sequence, series, dict value;`fill_value`: if specified then it fills the na value by this default value.|
|mul, *| arithematic |Get multiplication of dataframe using elementwise binary operation *. The operation is equivalent to dataframe * other [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.mul.html)| `source_cols`: source columns on which the dataframe operation needs to be performed;`dest_cols`: place where to store results;`operands`: scalar, sequence, series, dict value;`fill_value`: if specified then it fills the na value by this default value.|
| mod, % | arithematic |Get modulo of dataframe using elementwise binary operation %. [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.mod.html) | `source_cols`: source columns on which the dataframe operation needs to be performed;`dest_cols`: place where to store results; `operands`: scalar, sequence, series, dict value;`fill_value`: if specified then it fills the na value by this default value.|
|pow, **| arithematic  | Get power of dataframe using elementwise binary operation ** [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.pow.html)| `source_cols`: source columns on which the dataframe operation needs to be performed; `dest_cols`: place where to store results; `operands`: scalar, sequence, series, dict value; `fill_value`: if specified then it fills the na value by this default value.|
|floordiv, // | arithematic | Get floordiv of dataframe using elementwise binary operation //. [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.floordiv.html)| `source_cols`: source columns on which the dataframe operation needs to be performed;`dest_cols`: place where to store results;`operands`: scalar, sequence, series, dict value;`fill_value`: if specified then it fills the na value by this default value.|
|min | arithematic | Calculates the minimum for columns passed. [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.min.html) | `source_cols`: source columns to consider; `dest_cols`: destination columns where to store the result; `axis`: integer indicating axis along which to compute the min value |
|max | arithematic | Calculates the maximum for columns passed. [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.max.html) | `source_cols:` source columns to consider; `dest_cols`: destination columns where to store the result; 
`axis`: Axis along which to compute the max value |
|median | arithematic | Calculates the median value for columns passed. [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.median.html) | `source_cols`: source columns to consider; `dest_cols`: destination columns where to store the result|
|std | arithematic | Calculates the standard deviation of columns passed. [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.std.html) | `source_cols`: source columns to consider; `dest_cols`: destination columns where to store the result;`ddof`: degrees of freedom for calculating the standard deviation|
|var | arithematic | Calculates the variance of columns passed [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.var.html)| `source_cols`: source columns to consider;`dest_cols`: destination columns where to store the result;`ddof`: Delta degrees of freedom|
| mean | arithematic | Calculates the mean of columns passed. [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.mean.html) | `source_cols`: source columns to consider;`dest_cols`: destination columns where to store the result |
|kurt | arithematic | Returns unbased kurtosis over the requested column or axis. kurtosis defines the tailedness of the probability distribution [description](https://en.wikipedia.org/wiki/Kurtosis) [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.kurt.html)| `source_cols:` source columns to consider;`dest_cols:` destination columns where to store the result|
| quantile | arithematic | Returns the value of given quantile over requested axis [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.quantile.html)| `source_cols`: source columns to consider;`dest_cols`: destination columns where to store the result;`quantile`: float or array like where the value of quantile is between 0 and 1;`interpolation_method`: The parameter specifies the interpolation method to use when the desired quantile lies between the 2 points the possible values are `linear`, `lower`, `higher`, `nearest`, `midpoint`;  `method`: 'single' or 'table' Whether to compute the quantiles per-column or over all columns When table, the only allowed interpolation are 'nearest', 'lower'and 'higher' |
| abs | arithematic | Returns a Series/Dataframe with the absolute metric value of each element. This function only applies to element that are all numeric.[refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.abs.html)| source_cols: source columns to consider ; `dest_cols`: destination columns where to store the result |
|clip | arithematic | Trims the values at input threshold; [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.clip.html) | `source_cols`: source columns where clipping needs to take place; `dest_cols`: Destination columns where the clipping results are stored;`lower`: Minimum threshold value below which the value. If A missing threshold then it will not clip the values;`upper`: Maximum threshold value above which to clip the value. If a missing threshold then it will not clip the value|
|corr | arithematic | pairwise correlation of the columns passed [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.corr.html) | `source_cols`: source columns to consider; `method`: Method of correlation. Allowed values include /`pearson`: standard correlation coefficient; `kendall`: Kendall Tau correlation coefficient;`spearman`: Spearman rank correlation|
| cov | arithemartic | Compute the pairwise covariance of columns, excluding NA/null values. Both NA and null values are automatically excluded from the calculation. A threshold can be set for the minimum number of observation for each value created; [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.cov.html) | `source_cols`: columns on which covariance is carried out. `min_period`: Minimum observations required. `ddof`: Delta degrees of freedom |
|nunique | arithematic | Calculates the unique value for columns. [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.nunique.html) |  `source_cols`: source columns to consider;`dest_cols`: destination columns where to store the result|
|count | arithematic | Counts non NA entries in the dataframe| `source_cols`: source columns to consider;`dest_cols`: destination columns where to store the result|
|sem | arithematic | Return unbaised standard error of the mean over requested axis [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.sem.html) |`source_cols`: source columns to consider; `dest_cols`: destination columns where to store the result;`skipna`: Skip Na; `ddof`: delta degree of freedom. The divisor is used in calculations of N-ddof where N represents the number of elements |
|skew | arithematic | Return unbaised skew over and axis [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.skew.html) | `source_cols`: source columns to consider;`dest_cols`: destination columns where to store the result;`skipna`: Skip Na |
|round | arithematic | Rounds a Dataframe to a variable number of decimal places [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.round.html) | `source_cols`: Columns on which the round operation occurs;`dest_cols`: If provided stores the result of the round operation;`decimal`: Number of places to round the results into|
|or, \ | arithematic | Performs boolean or operation [refer](https://pandas.pydata.org/docs/user_guide/indexing.html#boolean-indexing) | `source_cols`: columns on which operation needs to be performed;`dest_cols`: columns where the results are stored; `operands`: list of parameters for which comparison is made |
|and, & | arithematic | Performs boolean and operation [refer](https://pandas.pydata.org/docs/user_guide/indexing.html#boolean-indexing) | `source_cols`: columns on which operation needs to be performed; `dest_cols`: columns where the results are stored;`operands`: list of parameters for which comparison is made |
|not, ! | arithematic | Performs boolean not operation [refer](https://pandas.pydata.org/docs/user_guide/indexing.html#boolean-indexing) | `source_cols`: columns on which operation needs to be performed;`dest_cols`: columns where the results are stored |
|eq, == | arithematic | Performs boolean equal operation [refer](https://pandas.pydata.org/docs/user_guide/indexing.html#boolean-indexing)| `source_cols`: columns on which operation needs to be performed;`dest_cols`: columns where the results are stored;`operands`: list of parameters for which comparison is made |
|ne, != | arithematic | Performs boolean not equal operation [refer](https://pandas.pydata.org/docs/user_guide/indexing.html#boolean-indexing) | `source_cols:` columns on which operation needs to be performed;`dest_cols:` columns where the results are stored;`operands:` list of parameters for which comparison is made |
|lt, < | arithematic | Performs boolean less than operation [refer](https://pandas.pydata.org/docs/user_guide/indexing.html#boolean-indexing) | `source_cols:` columns on which operation needs to be performed;`dest_cols:` columns where the results are stored;`operands:` list of parameters for which comparison is made|
|gt, > | arithematic | Performs boolean greater than operation [refer](https://pandas.pydata.org/docs/user_guide/indexing.html#boolean-indexing) | `source_cols:` columns on which operation needs to be performed;`dest_cols:` columns where the results are stored;`operands:` list of parameters for which comparison is made |
|lte, <= | arithematic | Performs boolean less than or equal operation [refer](https://pandas.pydata.org/docs/user_guide/indexing.html#boolean-indexing) | `source_cols:` columns on which operation needs to be performed;`dest_cols:` columns where the results are stored;`operands:` list of parameters for which comparison is made |
|gte, >= | arithematic | Performs boolean greater than or equal operation [refer](https://pandas.pydata.org/docs/user_guide/indexing.html#boolean-indexing) | `source_cols:` columns on which operation needs to be performed;`dest_cols:` columns where the results are stored;`operands:` list of parameters for which comparison is made|
|basic_ops | arithematic | Inspired from pandas eval function perform combination of airthematic operations on the set of columns. operation can be single function or a set of instruction. The function expects data structure operation of following format. Grammar of [Basic Ops] | `source_cols`: Source data columns on which the operations are performed, `dest_cols`: Columns where the results are stored. `operations`: dictionary containing the key, value of the operation to be performed  |
|remove_duplicates| common | drops duplicates from columns under consideration [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.drop_duplicates.html) | `source_cols`: columns where duplicate data is dropped; `keep`: first: Drop the duplicates for the first occurance; `last`: Drop the dplicates for the last occurrence; `False`: Drop all duplicates|
|copy, copy_contents | common | Copy data from source columns to destination columns [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.copy.html) | `source_cols:` source columns where copy needs to be done;`dest_cols:` destination columns where data is copied |
|rename | common | renames the columns in dataframe [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.rename.html#) | `source_col`: source column name; `dest_col`: new column name |
|fillna | common | Fills NA values for the dataframe [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.fillna.html) | `source_cols`: source column names; `dest_cols`: if specified the replaced values are stored here; `replace`: value by which to replace the data by|
|add_prefix | common | Adds prefix on the column names for the existing dataframe [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.add_prefix.html) | `source_cols`: Columns on which prefixing is required; `prefix`: prefix to be applied|
|add_suffix | common | Adds column suffix to the existing column names [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.add_suffix.html) | `source_cols`: Columns on which prefixing is required; `suffix`: suffix to be applied|
|duplicated | common | Indicates whether a particular row is duplicated [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.duplicated.html) | `source_cols`: column name where duplication is tested; `dest_cols`: columns where the results are stored.`keep`: `first` Drop the duplicates for the first occurance `last` Drop the dplicates for the last occurrence `False` Drop all duplicates |
| sort | common | Sorts the value along columns [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.sort_values.html) | `cols`: str/list of columns by which to sort the data;`na_position`: where to position NA values |
| nlargest | common | returns first n rows ordered by columns in descending order  [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.nlargest.html) | `source_cols`: subset of columns; `nrows`: number of rows to return;`keep`: how to manage the duplicated values `first`,`last`,`all`|
| nsmallest | common | returns first n smallest rows orded by columns [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.nsmallest.html) | `source_columns`: subset of columns; `nrows`: number of rows to return; `keep`: how to manage duplicated data |
| shift | common | Shift Index by desired number of periods with an optional time freq.  When the freq is not passed, shift the index without realigning the data. If the freq is passed, the index will be increased using the periods and the freq . freq can be inferred when specified as 'infer' as long as either freq or inferred_freq attribute is set in the index [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.shift.html) | `source_cols`: source columns used for shifting;`dest_cols`: (optional) destination columns where the results are stored; `periods`: Number of periods/frequency to shift; `axis`: shift direction; `fill_value`: scalar value for newly introduced value |
| split | common | For a list/tuple or a set like data structure the function flattens the data into columns. Column names are `col_{pos}` optionally prefix can be specified for naming structure like `col_{prefix}_{pos}`. Limitation: current function only takes in string column | `source_col`: column Name under consideration; `prefix`: column naming prefix appended to the column |
| split_rows | common | For a list/tuple or a set like data structure the function explodes the into a row. The function will replicate the index values params: source_column: IndexLabel Column(s) to explode. For multiple columns, specify a non empty list each element be str or tuple and all specified columns their list-like data on the same row must have the matching length ignore_index: bool; default False;If set to true the resulting label will be set to 0,1,..n-1; return_dest: If set to true will return the original dataframe |
| assign | common | Replace all cell values on the mentioned columns for the existing dataframe. The function only supports one to one column mapping. The function supports list, dictionary and tuple assignment; however these datatypes are not recommended as it reduces vectorized operations for details [check](https://stackoverflow.com/questions/993984/what-are-the-advantages-of-numpy-over-regular-python-lists) | `source_cols`: Columns on which all of its cell values need to replace. `replace`: Value by which to replace the data by |
| conditional_concat | common | Allows conditional concatenating of the columns to a destination string. This function allows specific string format to be appended to the destination column name `params:`; `format_string`: f string formatted string to define the pattern for concatenation;`source_cols`: list of source column or regex expression representing source and destination columns;`dest_cols`: column name where the result is stored;`regex_match`: Allows loop based regex matching. If set to true it assumes source_cols parameter representing a regex expression it will perform string concatenation for all the set of matching expression `sep`: valid only if regex is true. The separator used for concatenating different strings; `append_column`: Boolean field to indicate whether to append column name at the start of each concatenation;`append_column_regex` : Boolean field to indicate whether the append column is a regex expression; `append_col_sep`:  Valid only if append_column_regex is true; to indicate how to split the column name;`append_column_sep_index`: Valid only if the append_column_regex is true  and append_col_sep is provided. Index value indicating the append_column value;`conditions`: list of dictionary or list of list of dictionary indicating the conditions on variable where format string should be valid. - simple list indicates the or condition;-  list of list indicates (or(combination of and condition)) conditions should match the source cols |
| to_numeric | type_conversion | Tries to convert data into numeric representation | `source_cols`: source columns which are subject to transformation; `dest_cols`: destination columns where converted results reside (optional) |
| to_datetime | type_conversion | Tries to convert data into datetime representation | `source_cols`: source columns which are subject to transformation; `dest_cols`: destination columns where converted results reside (optional) |
| to_string | type_conversion | Convert the datatype of the columns to string | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored|
| to_float | type_conversion | Convert the datatype of the columns to float  |`source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| to_integer | type_conversion | Convert the datatype of the columns to integer |`source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| to_boolean | type_conversion | Convert the datatype of the columns to boolean | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| infer | type_conversion | Infers the datatype and converts it into inferred data type | Convert the datatype of the columns to boolean | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| coalesce | evaluate | Splunk function: Takes one or more values and returns the first not NULL value [refer](https://www.splunk.com/en_us/blog/tips-and-tricks/search-command-coalesce.html) | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| md5 | hash | Converts String into md5 equivalent | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| sha1 | hash | Converts String into sha1 equivalent | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| sha224 | hash | Converts String into sha224 equivalent | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| sha256 | hash | Converts String into sha256 equivalent | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| sha384 | hash | Converts String into sha384 equivalent | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| sha512 | hash | Converts String into sha512 equivalent | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| sha3_224 | hash | Converts String into sha3_224 equivalent | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| sha3_256 | hash | Converts String into sha3_256 equivalent | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| sha3_384 | hash | Converts String into sha3_384 equivalent | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| sha3_512 | hash | Converts String into sha3_512 equivalent | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| shake_128 | hash | Converts String into shake_128 equivalent | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored; `length`: length of the hex digest to return |
| shake_256 | hash | Converts String into shake_128 equivalent | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored; `length`: length of the hex digest to return |
| to_ipaddress | ipaddess_ipnetwork | Converts string or integer object into ipaddress object | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| to_ipnetwork | ipaddess_ipnetwork | Converts to ipnetwork object| `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| to_ipinterface | ipaddess_ipnetwork | Converts the the data into ip interface class | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipaddress_to_int | ipaddess_ipnetwork | Returns integer equivalent for ipaddress | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipaddress_to_float | ipaddess_ipnetwork | Returns floating point equivalent for ipaddress | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipaddress_version | ipaddess_ipnetwork | Returns the version of ipnetwork object | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipaddress_reverse_pointer | ipaddess_ipnetwork | Returns the reverse data pointer fo | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipaddress_multicast | ipaddess_ipnetwork | Return if ip_address is used for multicast use | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipaddress_is_private | ipaddess_ipnetwork | If the ipaddress is private | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipadress_is_global | ipaddess_ipnetwork | If ipaddress is global | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipaddress_is_unspecified | ipaddess_ipnetwork | If ipaddress is unspecified | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipaddress_is_reserved | ipaddess_ipnetwork | If ipaddress is reserved | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipaddress_is_link_local | ipaddess_ipnetwork | If ipaddress is link local address | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipaddress_is_sitelocal | ipaddess_ipnetwork | If ipaddress is a site local | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipnetwork_version | ipaddess_ipnetwork | Returns the ip network version | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipnetwork_is_multicast | ipaddess_ipnetwork | Returns if the network is multicast | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipnetwork_is_private | ipaddess_ipnetwork | Checks if the network is private | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipnetwork_is_unspecified | ipaddess_ipnetwork | If network is unspecified | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipnetwork_is_reserved | ipaddess_ipnetwork | Network is reserved | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipnetwork_is_loopback | ipaddess_ipnetwork | Network is a loopback network | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| ipnetwork_is_linklocal | ipaddess_ipnetwork | Network is a linklocal network | `source_col`: source columns containing the data;`dest_col`: optional where the result of the operation is stored |
| network_address | ipaddess_ipnetwork | Returns the address component for the network | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| broadcast_address | ipaddess_ipnetwork | Returns broadcast address | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| network_hostmask | ipaddess_ipnetwork | Returns hostmast for the ip network | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| network_netmask | ipaddess_ipnetwork | Returns the netmask for the ip network | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| network_prefixlen | ipaddess_ipnetwork | Returns the prefixlen for the network | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| network_numaddresses | ipaddess_ipnetwork | Returns the number of ipaddresses in the network | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| network_hosts | ipaddess_ipnetwork | It returns a list of hosts in a newtwork  |  `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored; `expand`: Whether to assign each ip address to a separate column |
| network_overlaps | ipaddess_ipnetwork | Check whether the networks overlaps | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored;`network`: String, list or column name representing the ip network.   |
| subnets | ipaddess_ipnetwork | Returns the subnets which when joined to make the current network. | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored;`prefixlen_diff`: The amount our prefix length should be increased by. `new_prefix`: is the desired new prefix of the subnets and must be larger than our current prefix,`expand`: whether to assign new column names for the subnets created |
| is_supernet | ipaddess_ipnetwork | Gets the supernet encompassing the current network | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored;`prefixlen_diff`: The amount by which to increase the prefix length by `new_prefix`: The desired new prefix length |
| is_subnet_of | ipaddess_ipnetwork | Check whether the network is a supernet of network2 | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored; `networks`: String, list or column name representation of ip networks or column containing the ip network |
| compare_networks | ipaddess_ipnetwork | For given network, it does a comparsion (less, greater or equal to)  for two network comparisons | `source_cols`: source columns containing the ip network; `dest_cols`: optional destination columns storing the comparison results;`networks`: ip network or column names to which comparison is made |
| group_to_network | ipaddess_ipnetwork | Groups IP address to IP network (v4 or v6 format) | `source_cols`: source columns storing the information;`dest_cols`: Optional destination column to store the results |
| network_address_membership | ipaddess_ipnetwork | For a given network checks it checks whether the ip address in the columns are a member of the network | `source_cols`: column names present in source dataframe; `dest_cols`: [optional] column where the results are stored; `ip_address`: IPv4Address, IPv6Address, column name or list combination of above |
| address_network_membership | ipaddess_ipnetwork | For a given ipaddress it checks whether it belongs to a column containing ip network | `source_cols`: column names present in source dataframe; `dest_cols`: [optional] column where the results are stored;`ip_address`: IPv4Network, IPv6Network, column name or list combination of above |
| combine_network | ipaddess_ipnetwork | Function merge the network having supernet subnet relationship | `source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| combine_to_supernet | ipaddess_ipnetwork | The function merges the ip network in the columns to a common supernet |`source_cols`:source columns whose types needs to be changed; `dest_cols`: destination columns where the results are stored |
| agg_ipaddress | ipaddess_ipnetwork | Defines the aggregation operation on dataframe for datatype belonging to class IP address. | `groupby_col`: columns representing the ip network, on which aggregation operation is carried out. `rename`: optional rename operation to be performed `merge`: boolean field indicating whether to merge back the aggregation result |
| transform_ip_addr | ipaddess_ipnetwork | Performs transformation function on grouped columns functions are specific for ipaddress and ip network | `groupby_col`: columns to group data on; `source_col`: columns toon which to perform aggregation function; `groupby_operation`: grouped/transformation operation to perform;`dest_cols`: columns where the data is stored |
| label_encoder_extn | label preprocessing | Label encoder extension class that provides long term encoding solution | `source_cols`: Source columns in the dataframe; `dest_cols`: Option destination datasets where the results are stored; `cust_id`: storing the customer id; `tenant_id`: tenant_id; `service_provider`: name of the csp name;`service_type`: name of the service_type |
| standard_scalar | label preprocessing | Performs standard scalar preprocessing | `source_cols`: Source column name in the dataframe. `dest_cols`: Destination column name in the dataframe |
| min_max | label preprocessing | Min Max scalar preprocessing it transforms the existing data column in min and max range values | `source_cols`: source columns in current dataframe;`dest_cols`: Where data is stored; `min_value`: Option value; `max_value`: Optional value |
| max_abs | label preprocessing | Scale each feature by its maximum absolute value/ This estimates scales and translates each feature individually such that the maximal absolute value of each feature in the training data set will be set  1.0. There is no shift/center the data and thus it does not destroy any sparsity |`source_cols`: Source columns needed transformation;`dest_cols`: where the results are stored |
| binarizer | label preprocessing | Binarize data according to a threshold (0 or 1) /Values greater than the threshold map to 1, values less than or equal ti the threshold map to 0. With the default threshold of 0, only positive values map to 1 ..Binarization is a common operation on text count data where the analyst can decide only to consider the presence or absence of a feature rather than quantified number of occurance for instance | `source_cols`: columns where the processing the is needed; `dest_cols`: where the preprocessing the data is stored;`threshold`: Feature values below or equal to this are replace by 0 above it by 1. Threshold may not be less than 0 for operations on sparse matrices.|
| kbins_discretizer | label preprocessing | Bin continuous data into interval. only ordinal is supported for now params | `source_cols`: source columns containing the data;`dest_cols`: destination columns where the encoded data is stored; `encode`: Method used to transform the data; `ordinal`: Return the bin identifier encoded as an integer value; `strategy`: `uniform`: All bins in each feature have identical widths `quantile`: All bins in each feature have the same number of points `kmeans`: Values in each bin have the same nearest center of 1D kmeans cluster; `random_state`: int; determines the number generation for subsampling. |
| label_encoder | label preprocessing | Encode target labels with values between 0  and n_classes-1. The transformation is useful for supervised learning and should encode the target value y and not the input X | `source_cols`: source columns under consideration;`dest_cols`: where the results are stored |
| normalizer | label preprocessing | Normalize samples individually to unit norm. Each sample with at least one non zero component is rescaled independently of other samples so that its norm equals to one. This transformer is able to work with both with dense numpy array and sparse matrix. Scaling inputs to unit norms is a common operation for text| `source_cols`: source columns under consideration;`dest_cols`: where the results are stored; `norm`: normalization method apply. Possible values are `l1`, `l2`, `max` |
| onehot_encoder | label preprocessing | Encode categorical features as one-hot numeric array The input to this transformer should be an array like integers or strings, denoting the values on by categorical features. The fetures are encoded using a one-hot encoding scheme, This creates a binary column for each category | `source_cols`: Source column names which contains the data; `categories`: `auto`: Determine categories automatically from training data, `list`: holds the categories expected in the ith column. The passes categories should not mux string and numeric values.`drop`: Specifies a methodology to use to drop one of the categories per feature.`sparse`: bool; Will return sparse matrix if set True else will return an array; `min_frequency`: Specifies the minimum frequency below which categories will be considered infrequent; `max_categories`: Specifies the upper limit to the number of output features for each input feature.|
| ordinal_encoder | label preprocessing | Encode categorical features as an interger array. The input to this transformer should be array like of integers or strings. Denoting the values taken on by categorical features. The features are converted into categorical ordinal integers | `source_cols`: Source columns that contains encoded data;`dest_cols`: Destination Columns where the results are stored; `categories`: Categories per feature; `encode_missing_value`: Encode value of missing categories;`min_frequency`: Specifies the minimum frequency below which a category will be considered infrequent; `max_categories`: Specifies the upper limit to the the number of categories|
| power_transform | label preprocessing | Apply power transform to feature wise to make the data more Gaussian-Like. Power transforms are a family of parametric, monotonic transformation that are applied to more Guassian-like. This transformation is more common for datasets where normalization is required.| `source_cols`: Column names which require transformation; `dest_cols`: Where results are stored. `method`: The power transformation methods Supported methods include: 'yeo-johnson': Works with positive and negative values 'box-cox': Only works with strictly positive values;`standardize`: Apply zero mean, unit variance normalization to the transformed |
| quantile_transform | label preprocessing | The transforma features using quantiles information. This method transforms the features to follow a uniform or a normal distribution For a given features, this transformation tends to spread out the most frequent values. Reduces the impact of outlier The transformation is applied on each feature independently.| `source_cols`: columns containing the data to be preprocessed; `dest_cols`: where the data is to be stored; `n_quantile`: Number of quantiles to be computed. It corresponds to the number of landmarks used to discretize the cumulative distribution function. `output_distribution`: Marginal dstributuin of the transformed data `uniform`, `normal`; `sub_samples`: Maximun number of samples used to estimate the quantile for computation efficiency `random_state`: Determines the random state used for number generation and smoothing noise |
| robust_scaler | label preprocessing | Scale features using statistics that are robust to outliers. This scaler removes the median and scales the data according to the quantile range. The IQR is the range between the 1st quartile (25th quantile) and the 3rd quartile (75th quantile) Centering and scaling happen independently on each feature by computing the relavant statistics on the samples in the training set | `source_cols`: column undergoing transformation; `dest_cols`: where the results of transformation is stored; `with_centering`: If true it will centre the data before the scaling. This will cause transform to raise the an exception when attempted in sparse matrices. `with_scaling`: If true, scale the data to interquantile range ; `quantile_range_min`: Range used to calculate IQR;`quantile_range_max`: Range used to calculate IQR; `unit_variance`: bool If true, scale data so that normally distributed features have a variance of 1.|
| to_upper | string operations | Converts given data to upper case | `source_cols`: Column representing data converted to upper case;`dest_cols`: Columns where the data is stored |
| to_lower | string operations | Converts the data to lower case | Column representing data converted to upper case;`dest_cols`: Columns where the data is stored|
| str_len | string operations | Returns the length of the data string | Column representing data converted to upper case;`dest_cols`: Columns where the data is stored|
| strip | string operations | Strip the data and removes the extra spaces | Column representing data converted to upper case;`dest_cols`: Columns where the data is stored|
| str_replace | string operations | Does the string replacement operation on data | `source_cols`: Column representing data converted to upper case;`dest_cols`: Columns where the data is stored; `source_pattern`: pattern that needs to be replaced; `destination_pattern`: Replacement to the current pattern; `regex`: boolean to indicate whether replacement would follow regex `case`: boolean to indicate whether to ignore case|
| str_split | string operations | Deoes a string split on string patterns. It replaces the string into an array of strings | `source_cols`: Column on which to splot operation. `split_on`: string pattern/patterns to split string on; `dest_col`: columns where to store the results. `num_splits`: int number of splits to carry on
;`orientation`: str, flag to indicate the direction from where we should start the split operation |
| get_timezone_equivalent | datetime operations | Converts the field into timezone equivalent representation. By default the library assumes that the date provided are in `utc` timezone | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same|
| num_days | datetime operations | Returns the number of days since the time period | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| num_months | datetime operations | Returns the number of months since the time period | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| num_years | datetime operations | Returns number of years for the respective field | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| year | datetime operations | Returns the year from the datetime field | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| month | datetime operations | Returns the year from the datetime field |`source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same|
| month_name | datetime operations | Get month name from the date passed | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| day | datetime operations | Returns the day from the datetime field | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| day_name | datetime operations | get the weekday name |  `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same  |
| week_day | datetime operations | Returns the week_day from the datetime field | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same|
| week | datetime operations | Return the week of the week | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same|
| year_day | datetime operations | returns day of the year |`source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| hour | datetime operations | Returns the hour from the datetime component | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same|
| minute | datetime operations | Returns minute component from the datetime component | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| second | datetime operations | Returns seconds component from the datetime| `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| time | datetime operations | Returns time component from the current timestamp | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| timestamp | datetime operations | Converts the data into timestamp representation | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| date | datetime operations | Returns ISO date represents of the current timestamp column | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| current_datetime | datetime operations | Assigns current time to the column | `dest_col`: column where the data is stored|
| quarter | datetime operations | Get the quarter representation from the current date  | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| is_month_start | datetime operations | Boolean to indicate whether its a month start | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| is_month_end | datetime operations | Boolean to indicate its month end | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| is_year_start | datetime operations | Boolean to indicate whether its year start | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| is_year_end | datetime operations | Whether the current date is year end | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| is_quarter_start | datetime operations | If the current date is quarter start | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| is_quarter_end | datetime operations | If the current time represents the quarter end | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| is_leap_year | datetime operations | Whether the current date represents a leap year | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| get_days_in_month | datetime operations | Get the number of days in the current month | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| normalize_date | datetime operations | normalizes the date to start of the date `00:00:00` | `source_col`: source columns; `dest_col`: destination columns; `timezone_name`: Timezone information to convert it into;`is_utc`: boolean to indicate whether the source field is in boolean format;`source_timezone`: source datetime timezone; `unit`: if datetime is in integer; then unit represents the integer representation for the same |
| offset_datetime | datetime operations | Adds or substract an offset from the current datetime representation. | `source_col`: source columns representing the datetime values; `dest_col`: destination columns representing the datetime value; `timezone`: Timezone information to convert the datetime into; `is_utc`: if the source column is in utc. `source_timezone`: if the source column is not in utc then the timezone of the source column; `offset_unit`:  Unit for offset value, allowed values [include](#time-delta-units); `offset_direction`: whether to rollforward or backward; `holiday_calendar`: custom holiday calendar, valid for custom business day. `week_mask`: custom week mask, valid for custom offset as week. `custom_business_hour_start`: custom business hour start, valid for business hour. `custom_business_hour_end`: custom business hour end, valid for business hour |
| timedetla | datetime operations | Timedelta: difference between the times expressed in different units e.g. days, hours, minutes, seconds. The difference would be either between two dataframe columns or timedelta function | `source_cols`: source columns for the timedelta. `dest_cols`: columns where the results are stored|
| calculate_timedelta | datetime operations | For a valid timedelta passed the function generates the resultant datetime | `source_cols`: source columns containing the datetime value. `dest_cols`: destination columns storing the result. `timedelta_unit`: string or list containing the timedelta unit. `unit`: tuni for converting the timestamp. `is_utc`: Is the source column in utc format. `source_timezone`: if not in utc this indicates the source timezone. |
| cumsum | rolling window | Performs cumulative sum on specified column [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.cumsum.html) | `source_cols`: columns in the source dataframe where cumsum operation takes place; `dest_cols`: columns where the resultant data is stored |
| cummin | rolling window | Performs cumulative min on specified column [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.cummin.html) | `source_cols`: columns in the source dataframe where cummax operation takes place; `dest_cols`: columns where the resultant data is stored |
| cummax | rolling window | Performs cumulative max on specified column [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.cummax.html) | `source_cols`: columns in the source dataframe where cummax operation takes place; `dest_cols`: columns where the resultant data is stored |
| cumprod | rolling window | Performs cumulative prod on specified column [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.cumprod.html) | `source_cols`: columns in the source dataframe where cumprod operation takes place; `dest_cols`: columns where the resultant data is stored |
| window | rolling window | Generic rolling window or variable window size over the values [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.rolling.html) | `source_cols`: column on which windowing operation is carried out. `dest_cols`: where the result of the windowing operation is stored. `period`: window size. `min_periods`: Minimum number of observation in window to have otherwise result is `np.nan`; `center`: Set the window to the center of the window index. `closed`: Which side of the interval is closed. Possible values incldue `right`: the first point in the window is excluded. `left`: the last point in the window is excluded. `both`: No points in window is excluded. `neither`: The first and last points in the window is excluded. `operation`: windowing operation to be performed. Supported window operations [include]|
| weighted_window | rolling window | Computes rolling function over a weighted window frame | `source_cols`: source columns containing the data. `dest_cols`: Column where the results are stored. `period`: Window size. `win_type`: Type of weighted operation. Supported weighted window [include](#weighted-window-type). `min_periods`: Mimimum window size. `closed`: How the windows are closed. Possible values include `right`, `left`,`both`, `neither`. operation: Possible windowing operation to be carried out [operations](#weighted-window-operations);`to_fill_na`: boolean to indicate whether to fill na. `fill_value`: if `to_fill_na` is true the the value will be used to fill na value. |
| expanding_window | rolling window | Provide an expanding window calculatuons [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.expanding.html) | `source_cols`: Columns on which expanding window operation is carried out. `dest_cols`: Columns where the results are stored. `min_periods`: minimum samples required for window based operations. `operation`: expanding window operation to be carried out. Supported operations [include](#expanding-window-operations) |
| exponential_window | rolling window | Provides an exponentially weighted operation [refer](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.ewm.html)|`source_cols`: column on which windownig operation is carried out. `dest_cols`: where the results of the windowing operation is stored. `period`: window size. `operation`: exponential window operation [supported] |
|common|isna| Returns True if the columns contain missing values. Returns a boolean same size object indicating if the values are NA. NA values such as None or numpy.NaN gets mapped to the True Values. | `source_cols`: Columns considered for checking NA values;  `dest_cols`: Columns where the results are stored;  `use_inf_as_na`: boolean indicating whether to consider some string such as '' or inf as NA values |
|common| isnotna| Detect existing (non-missing) values. Return a boolean same-size object indicating if the values are not NA. Not missing values get mapped to True.|`source_cols`: Columns considered for checking not NA values;  `dest_cols`: Columns where the results are stored;  `use_inf_as_na`: boolean indicating whether to consider some string such as NA |
| common | split_dict | If a data column contains dictionary into separate columns. `column_name` will be the key and value being the cell value/ Note this assumes, data values are hashmap; For non hashmap based values operation would return a empty dataframe.Operation performance would decrease if the key values are not consistent across the rows.| `source_cols`: columns containing dictionary/hashmap data; `prefix`: column prefix. Should match source_cols length; else it defaults to column_name; `drop_cols`: boolean value to indicate whether to drop the column containing hash value|
| common | drop_cols | Drops the columns from the dataframe | `source_cols`: columns to be dropped from the dataframe |

### Aggregate functions

| Function Name | Description |
| ------------- | ----------- |
| count | Returns the number of non-null values in the group |
| size  | Returns the number of rows |
| mean | Mean values over the reuqested axis |
| median | Median values over the requested axis |
| std | Standard deviation over the requested axis |
| var | Variance over the requested axis |
| sem | Standard error of the mean over the requested axis |
| min | Minimum values over the requested axis |
| max | Maximum values over the requested axis |
| first | First values over the requested axis |
| last | Last values over the requested axis |
| unique | Unique values over the requested axis. Returns the list of these values |
| list | List of values over the requested axis |
| nunique | Number of unique values over the requested axis |
| mode | Mode values over the requested axis |
| prec | Percentile values over a particular series |
| upperprec | Upper percentile values (0.75) over a particular axis |
| lowerprec | Lower percentile values (0.25) over a particular axis |
| nearestperc | Nearest percentile values (0.5) over a particular axis |
| midpointperc | Midpoint percentile values (0.5) over a particular axis |
| sumsquare | Sum of squares over the requested axis |

### Time Based Aggregate Operations

| Function Name | Description |
| ------------- | ----------- |
| sum           | Sum of the values over the requested axis |
| mean          | Mean of the values over the requested axis |
| avg           | Average of the values over the requested axis |
| std           | Standard deviation of the values over the requested axis |
| sem           | Standard error over mean over a particular axis |
| max           | Returns maximum value over a particular axis |
| min           | Returns minimum value over a particular axis |
| median        | Returns median value over a particular axis |
| first         | Returns first value over a particular axis |
| last          | Returns last value over a particular axis |
| last          | Returns last value over a particular axis |
| ohlc          | For a particular time period it returns the open, high, low and close values |
| count         | Returns the number of non-null values over a particular axis |
| size          | Returns the number of rows over a particular axis |
| nunique       | Returns the number of unique values over a particular axis |

### Basic Ops

Grammar of basic ops

```json
operations = [
    {
        operation: (operand, **extra_params) # case 1
        operation: (source_cols, operand, **extra_params) # case 2
    }
]
```

Grammar description

1. Case1:
   `Operation` is the operation to be performed (structure of the data is in postfix fashion). Operand is the argument for the binary operation.
   `extra_params`: dictionary containing the key value parameters for the operation to be performed.
2. Case 2:
   `source_cols`: Source data columns on which the operations are performed. `operand`: Operands on which the operation is carried out.
   `dest_cols`: Destination columns in which the results are stored (if supplied)
   `extra_params`: dictionary containing extra parameter for the function.

### Time-delta units

| Unit | Description |
| ---- | ----------- |
| b,bday, businessday, business_day | Uses default business day component |
| c, custombusinessday, custom_business_day | Uses custom business day component |
| d, day, days | Calendar day |
| w, week, weekly | weekly |
| wom, weekofmonth | the x-th day of the y-th week of each month |
| lwom, lastweekofmonth | The x-th day of the last week of each month |
| m, month, monthend | calendar month end |
| ms, monthstart, monthbegin | calendar month begin |
| bm, bmonthend, businessmonthend | business month end |
| bms, bmonthbegin, businessmonthbegin | business month begin |
| cbm, cbmonthend, custombusinessmonthend | custom business month end |
| cbms, cbmonthbegin, custombusinessmonthbegin | custom business month begin |
| sm, semimonthend, semi_month_end | 15th (or other day_of_month) and calendar month end |
| sms, semi_month_begin, semimonthbegin | 15th (or other day_of_month) and calendar month begin |
| q, quarter, quarter_end, qtr, qend | calendar quarter end |
| qs, quarter_start, quarter_begin, qstart, qbegin | calendar quarter begin |
| bq, bquarter_end, businessquarter_end, bqtr, bquarter_end | business quarter end |
| a, year_end, annual, annual_end | calendar year end |
| as, year_start, annual_start | calendar year begin |
| ba, byear_end, businessyear_end | business year end |
| bh, business_hour, bh, businesshour | business hour |
| h, hour, hours | hour |
| t, minute, min, minutes | minute |
| s, second, sec, seconds | second |

### Window operations

| Operation | Description |
| --------- | ----------- |
| count     | count values per window |
| sum       | sum of value per window |
| mean     | mean of values per window|
| median     | median of values per window|
| var     | variance of values per window|
| std     | standard deviation of values per window|
| min     | minimum values per window|
| max     | maximum values per window|
| corr     | correlation of values per column per window|
| skew     | skew values per column per window|
| kurt     | Returns unbased kurtosis over the requested column per window|
| quantile     | Returns quantile per column per window|
| sem     | Return unbaised standard error of the mean over requested axis per window|
| rank | Computes numerical data ranks (1 through n) along axis per window|

### Weighted window type

| window type | Description |
| ----------- | ----------- |
| barthann    | Bartlett-Hann window |
| bartlett    | Bartlett window |
| blackman    | Blackman window |
| blackmanharris | Blackman-Harris window |
| bohman      | Bohman window |
| boxcar      | Boxcar window |
| chebwin     | Chebyshev window |
| cosine      | Cosine window |
| dpss        | Siscrete Prolate Spheroidal Sequences |
| exponential | Exponential window |
| flattop     | Flat top window |
| gaussian    | Gaussian window |
| general_gaussian | Generalized Gaussian window |
| general_hamming | Generalized Hamming window |
| hamming     | Hamming window |
| hann        | Hann window |
| kaiser      | Kaiser window |
| kaiser_bessel_window | Kaiser-Bessel derived window |
| lanczos     | Lanczos window |
| nuttall     | Nuttall window |
| parzen      | Parzen window |
| taylor     | Taylor window |
| triang      | Triangular window |
| tukey       | Tukey window |


### Weighted window operations

| Operation | Description |
| --------- | ----------- |
| mean      | Weighted mean per window |
| sum       | Weighted sum per window |
| var      | Weighted variance per window |
| std      | Weighted standard deviation per window |

### Expanding window operations

| Operation | Description |
| --------- | ----------- |
| count     | count values per window |
| sum       | sum of value per window |
| mean     | mean of values per window|
| median     | median of values per window|
| var     | variance of values per window|
| std     | standard deviation of values per window|
| min     | minimum values per window|
| max     | maximum values per window|
| corr     | correlation of values per column per window|
| skew     | skew values per column per window|
| kurt     | Returns unbased kurtosis over the requested column per window|
| quantile     | Returns quantile per column per window|
| sem     | Return unbaised standard error of the mean over requested axis per window|
| rank | Computes numerical data ranks (1 through n) along axis per window|

### Exponential window operations

| Operation | Description |
| --------- | ----------- |
| sum       | sum of value per window |
| mean     | mean of values per window|
| std     | standard deviation of values per window|
| var     | variance of values per window|
| corr     | correlation of values per column per window|
| cov     | Covariance of the values per column per window |



## Appendix

Few important points

- For generic detection execution, especially for detection_type simple or complex-select, following precendence is followed for preparing rule filters:
  `time_diff` > `time_start_relative`, `time_end_relative` > `time_start`, `time_end`

- `Vertical Joins` consider column name as a key for joining the data and currently only support `inner`, `left`, `right` and `outer`

### Join Types

| Join Type | Join Direction | Description | Example |
| --------- | -------------- | ----------- | ------- |
| inner     | horizontal     | Performs SQL style inner joins for the dataframe under consideration |
| left      | horizontal     | Performs SQL style left joins for the dataframe under consideration |
| right     | horizontal     | Performs SQL style right joins for the dataframe under consideration |
| outer     | horizontal     | Performs SQL style outer joins for the dataframe under consideration |
| inner     | vertical       | Concatenates data frames. Only columns with common names are considered and concatenated. The both the dataframes have none of the columns this would result in an empty dataframe  |
| left      | vertical       | Concatenates dataframe keeping the columns from left dataframe intact |
| right     | vertical      | Concatenates dataframe keeping the columns from right dataframe intact |
| outer     | vertical       | Concatenates data frames. All columns are considered and concatenated. For a column name non existant in either of the joining data will its data replaced by NA for the corresponding column |


