# Porusenie vysielasich regulacii koncovym zariadenim
- Legitimne koncove zariadenia odosielaju data kazdych 30 sekund, co predstavuje 2880 odoslanych sprav denne
- Koncove zariadenia su jednoznacne identifikovane na zaklade deviceInfo.devEui
- Vytvorili sme pravidlo, ktore agreguje spravy za poslednych 24hodin a identifikuje zariadenia odosielajuce vacsi pocet sprav ako je 3 000 a v pripade identifikacie zariadenia porusujuceho regulacie generuju upozornenie

## Pravidlo pre ELK SIEM
```json
{"id":"5aff3150-ddbb-11ed-a1ab-5dee06af8c3c","updated_at":"2023-04-18T07:35:26.944Z","updated_by":"elastic","created_at":"2023-04-18T07:33:47.854Z","created_by":"elastic","name":"COUNT","tags":[],"interval":"24h","enabled":true,"description":"Rule for checking packet count for specific devEUI","risk_score":69,"severity":"medium","license":"","output_index":"","meta":{"from":"24h","kibana_siem_app_url":"http://10.0.130.124:5601/app/security"},"author":[],"false_positives":[],"from":"now-172800s","rule_id":"50c64f63-74b0-40b0-984c-9c50079e70fb","max_signals":100,"risk_score_mapping":[],"severity_mapping":[],"threat":[],"to":"now","references":[],"version":2,"exceptions_list":[],"immutable":false,"related_integrations":[],"required_fields":[],"setup":"","type":"threshold","language":"kuery","data_view_id":"b0505ffb-65d2-4f15-9e88-be6cf29be84c","query":"*","filters":[],"threshold":{"field":["deviceInfo.devEui.keyword"],"value":3000,"cardinality":[]},"throttle":"no_actions","actions":[]}
{"exported_count":1,"exported_rules_count":1,"missing_rules":[],"missing_rules_count":0,"exported_exception_list_count":0,"exported_exception_list_item_count":0,"missing_exception_list_item_count":0,"missing_exception_list_items":[],"missing_exception_lists":[],"missing_exception_lists_count":0}
```
