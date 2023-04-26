# Anomalie v datovych hodnotach
- Koncove zariadenia simuluju meranie teploty v laboratoriu v ktorom teplota ma byt v rozmedzi 15-30 stupnov Celzia
- Data o teplote su z koncovych zariadeni odosielane vo formate JSON
- Vytvorili sme pravidlo, identifikujuce anomaliu v sledovanych datach a v pripade ak teplota nie je z rozmedzia moznych teplot nastane vygenerovanie upozornenia


## Pravidlo pre ELK SIEM
```json
{"id":"886a4750-ddbc-11ed-a1ab-5dee06af8c3c","updated_at":"2023-04-26T08:49:23.933Z","updated_by":"elastic","created_at":"2023-04-18T07:42:13.320Z","created_by":"elastic","name":"TEMP","tags":[],"interval":"1m","enabled":true,"description":"Rule for checking the temperature range and format","risk_score":73,"severity":"high","license":"","output_index":"","meta":{"from":"55s","kibana_siem_app_url":"http://10.0.130.124:5601/app/security"},"author":[],"false_positives":[],"from":"now-115s","rule_id":"70d52d03-bf0b-40de-809e-1e55645e8ca3","max_signals":100,"risk_score_mapping":[],"severity_mapping":[],"threat":[],"to":"now","references":[],"version":6,"exceptions_list":[],"immutable":false,"related_integrations":[],"required_fields":[],"setup":"","type":"query","language":"kuery","data_view_id":"b0505ffb-65d2-4f15-9e88-be6cf29be84c","query":"data-unpacked.temp.keyword < 15 or data-unpacked.temp.keyword > 35 or NOT (data-unpacked.temp.keyword is number)","filters":[],"throttle":"rule","actions":[{"group":"default","id":"6d4d7dc0-cd31-11ed-a1ab-5dee06af8c3c","params":{"documents":[{"fuck":"me"}]},"action_type_id":".index"}]}
{"exported_count":1,"exported_rules_count":1,"missing_rules":[],"missing_rules_count":0,"exported_exception_list_count":0,"exported_exception_list_item_count":0,"missing_exception_list_item_count":0,"missing_exception_list_items":[],"missing_exception_lists":[],"missing_exception_lists_count":0}
```
