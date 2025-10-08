# IAM Roles System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          AWS LAMBDA FUNCTION                             │
│                       IAM Roles Management API                           │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         ENTRY POINT LAYER                                │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ lambda_function.py                                                │  │
│  │  • Receives HTTP event from API Gateway                          │  │
│  │  • Extracts HTTP method & query parameters                       │  │
│  │  • Validates caller identity                                     │  │
│  │  • Routes to appropriate handler                                 │  │
│  │  • Returns formatted HTTP response                               │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          HANDLERS LAYER                                  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ handlers/role_handler.py                                          │  │
│  │                                                                   │  │
│  │  handle_options_request()     → CORS preflight                   │  │
│  │  handle_post_request()        → Create role                      │  │
│  │  handle_get_request()         → Retrieve role(s)                 │  │
│  │  handle_put_request()         → Update role / user override      │  │
│  │  handle_delete_request()      → Delete role                      │  │
│  │  extract_caller_identity()    → Auth validation                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         SERVICES LAYER                                   │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ services/                                                         │  │
│  │                                                                   │  │
│  │  role_creation_service.py                                        │  │
│  │    • Validate permissions (IAM.create)                           │  │
│  │    • Parse & normalize policies                                  │  │
│  │    • Check for conflicts                                         │  │
│  │    • Generate IDs & store in DB                                  │  │
│  │                                                                   │  │
│  │  role_retrieval_service.py                                       │  │
│  │    • Query policy engine for allowed IDs                         │  │
│  │    • Fetch from database                                         │  │
│  │    • Filter based on permissions                                 │  │
│  │    • Format responses                                            │  │
│  │                                                                   │  │
│  │  role_update_service.py                                          │  │
│  │    • Validate update permissions                                 │  │
│  │    • Merge policy changes                                        │  │
│  │    • Update database atomically                                  │  │
│  │                                                                   │  │
│  │  role_deletion_service.py                                        │  │
│  │    • Check delete permissions                                    │  │
│  │    • Soft delete (status change)                                 │  │
│  │    • Hard delete (with cascade)                                  │  │
│  │                                                                   │  │
│  │  user_customization_service.py                                   │  │
│  │    • Validate user & base role                                   │  │
│  │    • Process module overrides                                    │  │
│  │    • Store in UserGrants table                                   │  │
│  │                                                                   │  │
│  │  user_role_view_service.py                                       │  │
│  │    • Fetch base role + overrides                                 │  │
│  │    • Merge policies intelligently                                │  │
│  │    • Return unified view                                         │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                     │                              │
                     ▼                              ▼
┌─────────────────────────────────┐  ┌──────────────────────────────────┐
│      MODELS LAYER                │  │   CROSS-CUTTING CONCERNS        │
│  ┌───────────────────────────┐  │  │  ┌────────────────────────────┐ │
│  │ models/                    │  │  │  │ policy_integration.py      │ │
│  │                            │  │  │  │  • can_do()                │ │
│  │  database.py              │  │  │  │  • get_allowed_record_ids()│ │
│  │    • DynamoDB connections │  │  │  │  • can_access_record()     │ │
│  │                            │  │  │  │  • Wraps policy_engine.py  │ │
│  │  role_repository.py       │  │  │  └────────────────────────────┘ │
│  │    • scan_all_roles()     │  │  │                                  │
│  │    • load_role_by_rid()   │  │  │  ┌────────────────────────────┐ │
│  │    • load_role_by_name()  │  │  │  │ policies.py                │ │
│  │    • batch_get_roles()    │  │  │  │  • normalize_policies()    │ │
│  │    • role_exists()        │  │  │  │  • deep_merge_policies()   │ │
│  │                            │  │  │  │  • has_any_allow()         │ │
│  │  assignment_repository.py │  │  │  └────────────────────────────┘ │
│  │    • load_user_assignments│  │  │                                  │
│  │    • list_users_by_role() │  │  │  ┌────────────────────────────┐ │
│  │    • validate_target_user │  │  │  │ overrides.py               │ │
│  │                            │  │  │  │  • get_user_overrides()    │ │
│  │  employee_repository.py   │  │  │  │  • process_override()      │ │
│  │    • get_employee_name()  │  │  │  │  • generate_ovid()         │ │
│  │                            │  │  │  └────────────────────────────┘ │
│  │  sequence_repository.py   │  │  │                                  │
│  │    • update_sequence()    │  │  │  ┌────────────────────────────┐ │
│  │    • get_display_id()     │  │  │  │ formatting.py              │ │
│  └───────────────────────────┘  │  │  │  • json_clean()            │ │
└─────────────────────────────────┘  │  │  • format_role_metadata()  │ │
                                     │  └────────────────────────────┘ │
                                     └──────────────────────────────────┘
                                                    │
                                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           UTILS LAYER                                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │ utils/                                                            │  │
│  │                                                                   │  │
│  │  response_utils.py    → build_response(), get_cors_headers()     │  │
│  │  validation_utils.py  → is_valid_user_id()                       │  │
│  │  time_utils.py        → now_iso()                                │  │
│  │  token_utils.py       → encode_token(), decode_token()           │  │
│  │  json_utils.py        → json_clean() for Decimal/sets            │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      EXTERNAL DEPENDENCIES                               │
│                                                                          │
│  ┌────────────────┐  ┌─────────────────┐  ┌──────────────────────┐   │
│  │  DynamoDB      │  │ Policy Engine   │  │  Configuration       │   │
│  │                │  │                 │  │                      │   │
│  │  • Roles Table │  │ policy_engine.py│  │  • config.py         │   │
│  │  • Grants Table│  │  (UNCHANGED)    │  │  • logging_config.py │   │
│  │  • Sequences   │  │                 │  │  • Environment vars  │   │
│  │  • Employees   │  │  • evaluate()   │  │                      │   │
│  └────────────────┘  │  • can_do()     │  └──────────────────────┘   │
│                      │  • Rules engine │                              │
│                      └─────────────────┘                              │
└─────────────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════

DATA FLOW EXAMPLE: GET /roles?rid=role-123

1. API Gateway → Lambda (lambda_function.py)
2. lambda_handler() extracts method=GET, rid=role-123
3. Routes to handle_get_request()
4. Handler calls handle_specific_role_view_by_rid()
5. Service calls can_access_record() → Policy Engine
6. Policy Engine evaluates permissions → ALLOW
7. Service calls load_role_by_rid() → Models
8. Models queries DynamoDB → Returns role
9. Service calls format_role_metadata() → Formatting
10. Service calls build_response() → Utils
11. Returns formatted HTTP response

═══════════════════════════════════════════════════════════════════════════

KEY PRINCIPLES APPLIED:

✅ Single Responsibility    Each module has one clear purpose
✅ Separation of Concerns   Layers don't leak into each other
✅ DRY (Don't Repeat)      No code duplication
✅ Dependency Injection     Services receive dependencies
✅ Interface Segregation    Clean package exports
✅ Open/Closed Principle    Easy to extend, hard to break

═══════════════════════════════════════════════════════════════════════════

DEPLOYMENT PACKAGE:

Required Files:
✅ lambda_function.py
✅ handlers/ (all files)
✅ services/ (all files)
✅ models/ (all files)
✅ utils/ (all files)
✅ policy_engine.py (unchanged)
✅ policy_integration.py
✅ policies.py
✅ overrides.py
✅ formatting.py
✅ config.py
✅ logging_config.py
✅ __init__.py

Documentation (optional):
📄 docs/OVERVIEW.md
📄 REFACTORING_SUMMARY.md
📄 QUICK_REFERENCE.md
📄 ARCHITECTURE.md (this file)

═══════════════════════════════════════════════════════════════════════════
```
