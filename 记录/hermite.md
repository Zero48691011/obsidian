```
INSERT INTO account_center.`user`
(id, email, password, name, phone, kind, status, create_time, update_time, phone_verify, oversea, nickname, channel, area_code, country_code, mini_program_appid, mini_program_scene, web_search_engine, utm_source)
VALUES(122538, 'lijiaxuan@dp.tech', 'pbkdf2:sha256:150000$BwxSvwGe$252c7ae59832f5d11ed4019bf6627cd5ab24e5c69ce3cb51c7c91ec79aec0097', '132****9161', '52D755AAB17C6A10183F9379B33BF71D', 2, 1, '2025-10-11 10:17:33', '2025-10-11 10:45:36', 1, 0, 'bohr6b12c4', 'pc', 86, '', '', 0, 'direct', '');


INSERT INTO account_center.organization
(id, name, full_name, org_type, sale_name, contact_name, contact_phone, tel_no, source, cert_no, remark, status, create_time, update_time)
VALUES(122395, '', '', 1, '', '', '', '', 4, '', '', 1, '2025-10-11 10:17:33', '2025-10-11 10:17:33');

INSERT INTO account_center.user_org_mapping
(id, user_id, org_id, is_org_owner, token, status, create_time, update_time)
VALUES(24286, 122538, 122395, 1, 'f094206efd0f4b00a392861558f3b9f1', 1, '2025-10-11 10:17:33', '2025-10-11 10:17:33')

```