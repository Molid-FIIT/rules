# Nedovoleny presun brany
- Existuje config, kde su zapisane "natvrdo" polohy brany
- Hodnota tohoto configu je potom jednym JSON presuvana na SIEM
- Je vytvorene pravidlo v ELK, ktore vygeneruje alert pri zmene hodnoty `lon` alebo `lat`

## Pravidlo pre ELK SIEM
```json
{"id":"6fa76650-dc66-11ed-a1ab-5dee06af8c3c","updated_at":"2023-04-16T14:53:23.849Z","updated_by":"elastic","created_at":"2023-04-16T14:53:23.849Z","created_by":"elastic","name":"GPS","tags":["gps"],"interval":"1m","enabled":true,"description":"Monitoring Gateway GPS coordinates from lora-kkt-fwd ;)","risk_score":21,"severity":"low","license":"","output_index":"","meta":{"from":"30s","kibana_siem_app_url":"http://10.0.130.124:5601/app/security"},"author":[],"false_positives":[],"from":"now-90s","rule_id":"e75b906d-b411-4d39-9eda-c92bc775b626","max_signals":100,"risk_score_mapping":[],"severity_mapping":[],"threat":[],"to":"now","references":[],"version":1,"exceptions_list":[],"immutable":false,"related_integrations":[],"required_fields":[],"setup":"","type":"new_terms","query":"*","new_terms_fields":["data.lat","data.lon"],"history_window_start":"now-1d","filters":[],"language":"kuery","data_view_id":"dba37bc6-deda-4629-be3b-87e54fbacb37","throttle":"rule","actions":[{"group":"default","id":"6d4d7dc0-cd31-11ed-a1ab-5dee06af8c3c","params":{"documents":[{"name":"lora_kkt_fwd_gps"}]},"action_type_id":".index"}]}
{"exported_count":1,"exported_rules_count":1,"missing_rules":[],"missing_rules_count":0,"exported_exception_list_count":0,"exported_exception_list_item_count":0,"missing_exception_list_item_count":0,"missing_exception_list_items":[],"missing_exception_lists":[],"missing_exception_lists_count":0}
```