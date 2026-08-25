# Full `process_action_type` catalog

Roughly 100 values on `process_action.process_action_type`, backed by the lookup entity
`process_action_type` (`process_action_type_name`, `system_flow_action` boolean,
`process_action_type_icon` enum `process_flow_action`/`system_flow_action`). Grouped by purpose below;
confirm the exact numeric value via `get_entity_definition`/`get_domain_definition` for the connector
in use before relying on a specific id, since these are illustrative of the catalog observed, not a
guaranteed-stable contract.

Legend: **sys** = `system_flow_action = true` (legal inside a system flow, no UI required); **ui** =
requires a live user session.

## Flow control

| Type | Value | | What it does |
|---|---|---|---|
| `start` | 98 | sys | The single entry node — a black connector, not a configurable action |
| `stop` | 99 | sys | Ends the flow (or one branch); a flow can have several |
| `decision` | 100 | sys | Routes based on process-variable conditions on its outgoing connectors; also the action to use for a code-only step (see main skill file) |
| `execute_user_sub_flow` | 761 | ui | Calls another process flow inline, interactively |
| `execute_system_sub_flow` | 760 | sys | Calls another system flow; supports async execution |
| `manual` | 0 | ui | Legacy no-op / status marker |

## Tasks & reports

| Type | Value | | What it does |
|---|---|---|---|
| `execute_tab_task` | 6 | ui | Runs a table task bound to a subject/table |
| `execute_task` | 60 | ui | Runs a standalone task, with parameter mapping to/from process variables |
| `execute_system_task` | 61 | sys | Runs a task's default procedure headless inside a system flow; parameters not marked input/output are ignored (model-validated) |
| `execute_tab_report` | 7 | ui | Starts a table report |
| `execute_report` | 70 | ui | Starts a standalone report |
| `generate_report` | 71 | sys | Produces a report headlessly (e.g. to attach to an email) instead of showing it |

## Table / grid / UI navigation (all **ui** — none legal in a system flow)

`activate_detail`(1), `activate_grid`(10), `activate_form`(11), `activate_form_list`(793),
`activate_card_list`(794), `activate_tree`(792), `activate_preview`(795), `activate_maps`(791),
`activate_scheduler`(790), `add_record`(3), `edit_record`(4), `delete_record`(5), `refresh_tab`(8),
`go_to_first_row`, `go_to_next_row`, `go_to_previous_row`, `go_to_last_row`, go-to-row-by-value,
`change_sorting`, `change_filter`, `change_prefilter`, `default_prefilters_on`, `clear_filters`,
`restore_filters`, `open_filter`, `zoom_in_on_detail`(22).

## Documents & misc UI

| Type | | What it does |
|---|---|---|
| `open_document`(2) | ui | Opens a document |
| `activate_document` | ui | Activates an already-open document |
| `close_document`(21) | ui | Closes a document |
| `close_all_documents`(870) | ui | Closes every open document |
| `open_link`(755) | ui | Opens an external URL |
| `copy_to_clipboard`(880) | ui | Copies a value/variable to the clipboard |
| `show_msg`(350) | ui | Shows a predefined message dialog, with variable substitution |
| `send_usr_notification`(351) | ui | Sends an in-app user notification (deep link, icon, expiry) |

## Connectors & integration (mostly **sys** — the actions that make a flow schedulable)

| Type | Value | What it does |
|---|---|---|
| `http_connector` | 600 | Calls a REST/HTTP endpoint; no top-level mandatory FK — config lives in child detail entities; being superseded by `web_connection` (a migration task `task_enrichment_conv_http_connector_to_web_connection` exists) |
| `web_connection` | 602 | Calls a configured Web connection endpoint; mandatory `web_connection_id` + `web_connection_endpoint_id` |
| `db_connector` | 590 | Runs a query against an external database connection |
| `ftp_connector` | 610 | FTP/SFTP file transfer |
| `smtp_connector` | 620 | Sends email via SMTP |
| `email_connector` | 625 | Sends email via a configured email provider |
| `appl_connector` | 580 | Calls another Thinkwise application |
| OAuth connector actions | 560/570/575 | Handle OAuth token flows for a configured OAuth server |
| `publish_message` | 950 | Publishes a message onto a configured message broker |

## Files (**sys**)

`download_file`(750), file-storage read/write/copy/move/delete (file-storage and non-file-storage
variants), `zip`/`unzip`, `merge_pdf`(860), `print_file`(770).

## Security & IAM (**sys**)

`encrypt`(780), `decrypt`(782), `hash_password`(110), `check_password`(111), `create_iam_user`(540),
`update_iam_user`(545), `create_usr_grp_assignment`(550), `update_usr_grp_assignment`(555).

## AI / ML (**sys**)

`llm_completion`(900), `llm_chat_completion`(901), `llm_instruction`(902), `llm_embedding`(903),
`automl_run_model`(850), `automl_train_classification_model`(820),
`automl_train_regression_model`(810), `automl_training_get_options`(800),
`timeseries_forecasting_connector`(830).

## Data transform & provisioning (**sys**)

`conv_json_to_xml`(700), `conv_xml_to_json`(710), `extract_json`(701), `provision_db`(585).

---

## Which types can be a starting point

Only these eight are ever wired as a `process_action_start_object_available` target in practice —
see the main skill file for the full explanation:

`activate_detail`(1), `open_document`(2), `add_record`(3), `edit_record`(4), `delete_record`(5),
`execute_tab_task`(6), `execute_task`(60), `execute_system_task`(61).

`start`/`stop` are excluded (they're the flow's own implicit entry/exit). Reports have schema support
(`report_id`/`report_variant_id` columns exist on `process_action_start_object_available`) but were
never observed populated in any live model sampled — treat a report-triggered start as
theoretically possible, not confirmed.

There is **no dedicated "control procedure" action type anywhere in this catalog** — control-procedure
logic always rides along inside an action via `use_processes`, or inside the task an
`execute_task`/`execute_system_task` action calls.
