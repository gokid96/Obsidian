### 1차 ERD 
![[Pasted image 20251106154231.png]]



초기 인프라 아키텍쳐
```
	 [사용자]
		↓
[도메인/DNS] (가비아, Route53 등) 
		↓ 
[AWS EC2] (단일 서버)
 ├─ Nginx (리버스 프록시, 80/443 포트) 
 ├─ Spring Boot (8080 포트) 
 └─ MySQL (3306 포트) 또는 AWS RDS
```

```

[1. 사용자 브라우저]
     ↓
병목: 이미지, CSS, JS 파일 다운로드 느림
해결: CDN 적용 (CloudFront) - 엣지 서버에서 정적 파일 제공
     ↓
     
[2. DNS 조회] (example.com → IP 변환)
     ↓
병목: 도메인 조회 느림
해결: 
  - Route53 같은 빠른 DNS 사용
  - DNS 캐싱 (브라우저, ISP 레벨)
     ↓
     
[3. EC2 Public IP / 네트워크]
     ↓
병목: EC2까지 네트워크 지연
해결: 
  - 가까운 리전 선택 (한국이면 ap-northeast-2)
  - Enhanced Networking 활성화
     ↓

┌─────────────────────────────────────┐
│  EC2 Instance                       │
│                                     │
│  [4. Nginx (Port 80, 443)]          │
│       ↓                             │
│  병목: SSL 처리 부하, 동시 연결 한계 │
│  해결:                              │
│    - worker_processes 증가          │
│    - worker_connections 증가        │
│    - keepalive_timeout 튜닝         │
│    - gzip 압축 활성화               │
│       ↓                             │
│                                     │
│  [5. Spring Boot (Port 8080)]       │
│       ↓                             │
│  병목: CPU/메모리 부족, 처리 느림    │
│  해결:                              │
│    - JVM Heap 메모리 조정           │
│    - Thread Pool 튜닝 (Tomcat)      │
│    - @Async 비동기 처리             │
│    - 로컬 캐시 적용 (Caffeine)      │
│    - 불필요한 로직 제거             │
│       ↓                             │
│                                     │
│  [6. Connection Pool (HikariCP)]    │
│       ↓                             │
│  병목: DB 연결 부족                 │
│  해결:                              │
│    - maximumPoolSize 조정           │
│    - connectionTimeout 튜닝         │
│       ↓                             │
│                                     │
│  [7. MySQL (Port 3306)]             │
│       ↓                             │
│  병목: 쿼리 느림, 동시 처리 한계     │
│  해결:                              │
│    - 인덱스 추가/최적화             │
│    - 슬로우 쿼리 개선               │
│    - N+1 문제 해결 (Fetch Join)     │
│    - 페이지네이션 최적화            │
│    - my.cnf 튜닝:                   │
│      · innodb_buffer_pool_size      │
│      · max_connections              │
│       ↓                             │
│                                     │
│  [8. 디스크 I/O]                    │
│       ↓                             │
│  병목: 디스크 읽기/쓰기 느림         │
│  해결:                              │
│    - EBS SSD로 변경 (gp3)           │
│    - IOPS 증가                      │
│    - 디스크 캐싱                    │
│                                     │
└─────────────────────────────────────┘

```





```
1단계: 기본 최적화 - 인덱스 추가 - 쿼리 최적화 (N+1 해결) - Connection Pool 튜닝 
2단계: 캐시 도입 - Redis 캐시 레이어 추가 - CDN 도입 (정적 파일) 
3단계: 수평 확장 - 애플리케이션 서버 증설 - 로드 밸런서 도입 - Read Replica 추가 
4단계: 고급 최적화 - DB 샤딩 - 비동기 처리 (메시지 큐) - MSA 전환
```








``` json
{

  "$schema": "https://raw.githubusercontent.com/dineug/erd-editor/main/json-schema/schema.json",

  "version": "3.0.0",

  "settings": {

    "width": 4000,

    "height": 3000,

    "scrollTop": -500,

    "scrollLeft": -560.1845,

    "zoomLevel": 0.76,

    "show": 431,

    "database": 4,

    "databaseName": "Astarchia",

    "canvasType": "ERD",

    "language": 4,

    "tableNameCase": 4,

    "columnNameCase": 2,

    "bracketType": 1,

    "relationshipDataTypeSync": true,

    "relationshipOptimization": false,

    "columnOrder": [

      1,

      2,

      4,

      8,

      16,

      32,

      64

    ],

    "maxWidthComment": -1,

    "ignoreSaveSettings": 0

  },

  "doc": {

    "tableIds": [

      "TABLE_USERS",

      "TABLE_POST",

      "TABLE_CATEGORY",

      "TABLE_SERIES",

      "TABLE_TAG",

      "TABLE_POSTTAG",

      "TABLE_IMAGE"

    ],

    "relationshipIds": [

      "REL_USER_POST",

      "REL_USER_CATEGORY",

      "REL_USER_SERIES",

      "REL_USER_IMAGE",

      "REL_CATEGORY_POST",

      "REL_SERIES_POST",

      "REL_POST_POSTTAG",

      "REL_TAG_POSTTAG"

    ],

    "indexIds": [],

    "memoIds": []

  },

  "collections": {

    "tableEntities": {

      "TABLE_USERS": {

        "id": "TABLE_USERS",

        "name": "Users",

        "comment": "회원",

        "columnIds": [

          "COL_USERS_ID",

          "COL_USERS_EMAIL",

          "COL_USERS_LOGIN_ID",

          "COL_USERS_PASSWORD",

          "COL_USERS_NICKNAME",

          "COL_USERS_CREATED_AT",

          "COL_USERS_UPDATED_AT"

        ],

        "seqColumnIds": [

          "COL_USERS_ID",

          "COL_USERS_EMAIL",

          "COL_USERS_LOGIN_ID",

          "COL_USERS_PASSWORD",

          "COL_USERS_NICKNAME",

          "COL_USERS_CREATED_AT",

          "COL_USERS_UPDATED_AT"

        ],

        "ui": {

          "x": 717.1053,

          "y": 25.0001,

          "zIndex": 1,

          "widthName": 60,

          "widthComment": 60,

          "color": "#F70505"

        },

        "meta": {

          "updateAt": 1762410817939,

          "createAt": 1762333724548

        }

      },

      "TABLE_POST": {

        "id": "TABLE_POST",

        "name": "Post",

        "comment": "게시글",

        "columnIds": [

          "COL_POST_ID",

          "COL_POST_USER_ID",

          "COL_POST_CATEGORY_ID",

          "COL_POST_SERIES_ID",

          "COL_POST_TITLE",

          "COL_POST_CONTENT",

          "COL_POST_SUMMARY",

          "COL_POST_THUMBNAIL_URL",

          "COL_POST_STATUS",

          "COL_POST_VISIBILITY",

          "COL_POST_VIEW_COUNT",

          "COL_POST_SLUG",

          "COL_POST_PUBLISHED_AT",

          "COL_POST_CREATED_AT",

          "COL_POST_UPDATED_AT"

        ],

        "seqColumnIds": [

          "COL_POST_ID",

          "COL_POST_USER_ID",

          "COL_POST_CATEGORY_ID",

          "COL_POST_SERIES_ID",

          "COL_POST_TITLE",

          "COL_POST_CONTENT",

          "COL_POST_SUMMARY",

          "COL_POST_THUMBNAIL_URL",

          "COL_POST_STATUS",

          "COL_POST_VISIBILITY",

          "COL_POST_VIEW_COUNT",

          "COL_POST_SLUG",

          "COL_POST_PUBLISHED_AT",

          "COL_POST_CREATED_AT",

          "COL_POST_UPDATED_AT"

        ],

        "ui": {

          "x": 676.2617,

          "y": 457.5883,

          "zIndex": 2,

          "widthName": 60,

          "widthComment": 60,

          "color": "#2196F3"

        },

        "meta": {

          "updateAt": 1762411191098,

          "createAt": 1762336073899

        }

      },

      "TABLE_CATEGORY": {

        "id": "TABLE_CATEGORY",

        "name": "Category",

        "comment": "카테고리",

        "columnIds": [

          "COL_CATEGORY_ID",

          "COL_CATEGORY_USER_ID",

          "COL_CATEGORY_NAME",

          "COL_CATEGORY_SLUG",

          "COL_CATEGORY_VISIBILITY",

          "COL_CATEGORY_DISPLAY_ORDER",

          "COL_CATEGORY_CREATED_AT",

          "COL_CATEGORY_UPDATED_AT"

        ],

        "seqColumnIds": [

          "COL_CATEGORY_ID",

          "COL_CATEGORY_USER_ID",

          "COL_CATEGORY_NAME",

          "COL_CATEGORY_SLUG",

          "COL_CATEGORY_VISIBILITY",

          "COL_CATEGORY_DISPLAY_ORDER",

          "COL_CATEGORY_CREATED_AT",

          "COL_CATEGORY_UPDATED_AT"

        ],

        "ui": {

          "x": 110.5263,

          "y": 244.7368,

          "zIndex": 3,

          "widthName": 60,

          "widthComment": 60,

          "color": "#4CAF50"

        },

        "meta": {

          "updateAt": 1762410993164,

          "createAt": 1762336073899

        }

      },

      "TABLE_SERIES": {

        "id": "TABLE_SERIES",

        "name": "Series",

        "comment": "시리즈",

        "columnIds": [

          "COL_SERIES_ID",

          "COL_SERIES_USER_ID",

          "COL_SERIES_NAME",

          "COL_SERIES_DESCRIPTION",

          "COL_SERIES_SLUG",

          "COL_SERIES_THUMBNAIL_URL",

          "COL_SERIES_VISIBILITY",

          "COL_SERIES_CREATED_AT",

          "COL_SERIES_UPDATED_AT"

        ],

        "seqColumnIds": [

          "COL_SERIES_ID",

          "COL_SERIES_USER_ID",

          "COL_SERIES_NAME",

          "COL_SERIES_DESCRIPTION",

          "COL_SERIES_SLUG",

          "COL_SERIES_THUMBNAIL_URL",

          "COL_SERIES_VISIBILITY",

          "COL_SERIES_CREATED_AT",

          "COL_SERIES_UPDATED_AT"

        ],

        "ui": {

          "x": 1522.3684,

          "y": 382.8949,

          "zIndex": 4,

          "widthName": 60,

          "widthComment": 60,

          "color": "#FF9800"

        },

        "meta": {

          "updateAt": 1762411001768,

          "createAt": 1762336073899

        }

      },

      "TABLE_TAG": {

        "id": "TABLE_TAG",

        "name": "Tag",

        "comment": "태그",

        "columnIds": [

          "COL_TAG_ID",

          "COL_TAG_NAME",

          "COL_TAG_SLUG",

          "COL_TAG_CREATED_AT"

        ],

        "seqColumnIds": [

          "COL_TAG_ID",

          "COL_TAG_NAME",

          "COL_TAG_SLUG",

          "COL_TAG_CREATED_AT"

        ],

        "ui": {

          "x": 1538.158,

          "y": 1002.6317,

          "zIndex": 5,

          "widthName": 60,

          "widthComment": 60,

          "color": "#9C27B0"

        },

        "meta": {

          "updateAt": 1762410973902,

          "createAt": 1762336073899

        }

      },

      "TABLE_POSTTAG": {

        "id": "TABLE_POSTTAG",

        "name": "PostTag",

        "comment": "게시글-태그 중간 테이블",

        "columnIds": [

          "COL_POSTTAG_ID",

          "COL_POSTTAG_POST_ID",

          "COL_POSTTAG_TAG_ID",

          "COL_POSTTAG_CREATED_AT"

        ],

        "seqColumnIds": [

          "COL_POSTTAG_ID",

          "COL_POSTTAG_POST_ID",

          "COL_POSTTAG_TAG_ID",

          "COL_POSTTAG_CREATED_AT"

        ],

        "ui": {

          "x": 1534.2105,

          "y": 705.2632,

          "zIndex": 6,

          "widthName": 60,

          "widthComment": 133,

          "color": "#795548"

        },

        "meta": {

          "updateAt": 1762410969645,

          "createAt": 1762336073899

        }

      },

      "TABLE_IMAGE": {

        "id": "TABLE_IMAGE",

        "name": "Image",

        "comment": "이미지",

        "columnIds": [

          "COL_IMAGE_ID",

          "COL_IMAGE_USER_ID",

          "COL_IMAGE_ORIGINAL_NAME",

          "COL_IMAGE_STORED_NAME",

          "COL_IMAGE_URL",

          "COL_IMAGE_SIZE",

          "COL_IMAGE_MIME_TYPE",

          "COL_IMAGE_CREATED_AT"

        ],

        "seqColumnIds": [

          "COL_IMAGE_ID",

          "COL_IMAGE_USER_ID",

          "COL_IMAGE_ORIGINAL_NAME",

          "COL_IMAGE_STORED_NAME",

          "COL_IMAGE_URL",

          "COL_IMAGE_SIZE",

          "COL_IMAGE_MIME_TYPE",

          "COL_IMAGE_CREATED_AT"

        ],

        "ui": {

          "x": 1518.4212,

          "y": 69.7368,

          "zIndex": 7,

          "widthName": 60,

          "widthComment": 60,

          "color": "#00BCD4"

        },

        "meta": {

          "updateAt": 1762411008306,

          "createAt": 1762336073899

        }

      }

    },

    "tableColumnEntities": {

      "COL_USERS_ID": {

        "id": "COL_USERS_ID",

        "tableId": "TABLE_USERS",

        "name": "user_id",

        "comment": "",

        "dataType": "BIGINT",

        "default": "",

        "options": 11,

        "ui": {

          "keys": 1,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762334292489,

          "createAt": 1762333729042

        }

      },

      "COL_USERS_EMAIL": {

        "id": "COL_USERS_EMAIL",

        "tableId": "TABLE_USERS",

        "name": "email",

        "comment": "",

        "dataType": "VARCHAR(100)",

        "default": "",

        "options": 12,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 81,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762335520122,

          "createAt": 1762334463005

        }

      },

      "COL_USERS_LOGIN_ID": {

        "id": "COL_USERS_LOGIN_ID",

        "tableId": "TABLE_USERS",

        "name": "login_id",

        "comment": "",

        "dataType": "VARCHAR(50)",

        "default": "",

        "options": 12,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 75,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762336042500,

          "createAt": 1762334463492

        }

      },

      "COL_USERS_PASSWORD": {

        "id": "COL_USERS_PASSWORD",

        "tableId": "TABLE_USERS",

        "name": "password",

        "comment": "",

        "dataType": "VARCHAR(255)",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 81,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762335546584,

          "createAt": 1762334463904

        }

      },

      "COL_USERS_NICKNAME": {

        "id": "COL_USERS_NICKNAME",

        "tableId": "TABLE_USERS",

        "name": "nickname",

        "comment": "",

        "dataType": "VARCHAR(50)",

        "default": "",

        "options": 12,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 75,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762335547078,

          "createAt": 1762334464737

        }

      },

      "COL_USERS_CREATED_AT": {

        "id": "COL_USERS_CREATED_AT",

        "tableId": "TABLE_USERS",

        "name": "created_at",

        "comment": "",

        "dataType": "TIMESTAMP",

        "default": "CURRENT_TIMESTAMP",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 65,

          "widthDefault": 122

        },

        "meta": {

          "updateAt": 1762335547566,

          "createAt": 1762335450645

        }

      },

      "COL_USERS_UPDATED_AT": {

        "id": "COL_USERS_UPDATED_AT",

        "tableId": "TABLE_USERS",

        "name": "updated_at",

        "comment": "",

        "dataType": "TIMESTAMP",

        "default": "CURRENT_TIMESTAMP",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 62,

          "widthComment": 60,

          "widthDataType": 65,

          "widthDefault": 122

        },

        "meta": {

          "updateAt": 1762335548038,

          "createAt": 1762335455101

        }

      },

      "COL_POST_ID": {

        "id": "COL_POST_ID",

        "tableId": "TABLE_POST",

        "name": "post_id",

        "comment": "",

        "dataType": "BIGINT",

        "default": "",

        "options": 11,

        "ui": {

          "keys": 1,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409594379,

          "createAt": 1762336405947

        }

      },

      "COL_POST_USER_ID": {

        "id": "COL_POST_USER_ID",

        "tableId": "TABLE_POST",

        "name": "user_id",

        "comment": "→ Users.user_id",

        "dataType": "BIGINT",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 2,

          "widthName": 60,

          "widthComment": 84,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_POST_CATEGORY_ID": {

        "id": "COL_POST_CATEGORY_ID",

        "tableId": "TABLE_POST",

        "name": "category_id",

        "comment": "→ Category.category_id",

        "dataType": "BIGINT",

        "default": "",

        "options": 0,

        "ui": {

          "keys": 2,

          "widthName": 63,

          "widthComment": 127,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409478692,

          "createAt": 1762336407349

        }

      },

      "COL_POST_SERIES_ID": {

        "id": "COL_POST_SERIES_ID",

        "tableId": "TABLE_POST",

        "name": "series_id",

        "comment": "→ Series.series_id",

        "dataType": "BIGINT",

        "default": "",

        "options": 0,

        "ui": {

          "keys": 2,

          "widthName": 60,

          "widthComment": 94,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409483091,

          "createAt": 1762409409220

        }

      },

      "COL_POST_TITLE": {

        "id": "COL_POST_TITLE",

        "tableId": "TABLE_POST",

        "name": "title",

        "comment": "",

        "dataType": "VARCHAR(200)",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 81,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409535544,

          "createAt": 1762409409367

        }

      },

      "COL_POST_CONTENT": {

        "id": "COL_POST_CONTENT",

        "tableId": "TABLE_POST",

        "name": "content",

        "comment": "",

        "dataType": "TEXT",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409536152,

          "createAt": 1762409409509

        }

      },

      "COL_POST_SUMMARY": {

        "id": "COL_POST_SUMMARY",

        "tableId": "TABLE_POST",

        "name": "summary",

        "comment": "",

        "dataType": "VARCHAR(300)",

        "default": "",

        "options": 0,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 81,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409498993,

          "createAt": 1762409409675

        }

      },

      "COL_POST_THUMBNAIL_URL": {

        "id": "COL_POST_THUMBNAIL_URL",

        "tableId": "TABLE_POST",

        "name": "thumbnail_url",

        "comment": "",

        "dataType": "VARCHAR(500)",

        "default": "",

        "options": 0,

        "ui": {

          "keys": 0,

          "widthName": 75,

          "widthComment": 60,

          "widthDataType": 81,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409503370,

          "createAt": 1762409409840

        }

      },

      "COL_POST_STATUS": {

        "id": "COL_POST_STATUS",

        "tableId": "TABLE_POST",

        "name": "status",

        "comment": "DRAFT/PUBLISHED",

        "dataType": "VARCHAR(20)",

        "default": "'DRAFT'",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 102,

          "widthDataType": 75,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409565812,

          "createAt": 1762409409990

        }

      },

      "COL_POST_VISIBILITY": {

        "id": "COL_POST_VISIBILITY",

        "tableId": "TABLE_POST",

        "name": "visibility",

        "comment": "PUBLIC/PRIVATE",

        "dataType": "VARCHAR(20)",

        "default": "'PUBLIC'",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 88,

          "widthDataType": 75,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409574725,

          "createAt": 1762409410148

        }

      },

      "COL_POST_VIEW_COUNT": {

        "id": "COL_POST_VIEW_COUNT",

        "tableId": "TABLE_POST",

        "name": "view_count",

        "comment": "DEFAULT 0",

        "dataType": "INT",

        "default": "0",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 61,

          "widthComment": 60,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409581919,

          "createAt": 1762409449448

        }

      },

      "COL_POST_SLUG": {

        "id": "COL_POST_SLUG",

        "tableId": "TABLE_POST",

        "name": "slug",

        "comment": "",

        "dataType": "VARCHAR(200)",

        "default": "",

        "options": 12,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 81,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409550194,

          "createAt": 1762409449862

        }

      },

      "COL_POST_PUBLISHED_AT": {

        "id": "COL_POST_PUBLISHED_AT",

        "tableId": "TABLE_POST",

        "name": "published_at",

        "comment": "",

        "dataType": "TIMESTAMP",

        "default": "",

        "options": 0,

        "ui": {

          "keys": 0,

          "widthName": 69,

          "widthComment": 60,

          "widthDataType": 65,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409525372,

          "createAt": 1762409450010

        }

      },

      "COL_POST_CREATED_AT": {

        "id": "COL_POST_CREATED_AT",

        "tableId": "TABLE_POST",

        "name": "created_at",

        "comment": "",

        "dataType": "TIMESTAMP",

        "default": "CURRENT_TIMESTAMP",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 65,

          "widthDefault": 122

        },

        "meta": {

          "updateAt": 1762409546554,

          "createAt": 1762409450162

        }

      },

      "COL_POST_UPDATED_AT": {

        "id": "COL_POST_UPDATED_AT",

        "tableId": "TABLE_POST",

        "name": "updated_at",

        "comment": "",

        "dataType": "TIMESTAMP",

        "default": "CURRENT_TIMESTAMP",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 62,

          "widthComment": 60,

          "widthDataType": 65,

          "widthDefault": 122

        },

        "meta": {

          "updateAt": 1762409545829,

          "createAt": 1762409464501

        }

      },

      "COL_CATEGORY_ID": {

        "id": "COL_CATEGORY_ID",

        "tableId": "TABLE_CATEGORY",

        "name": "category_id",

        "comment": "",

        "dataType": "BIGINT",

        "default": "",

        "options": 11,

        "ui": {

          "keys": 1,

          "widthName": 63,

          "widthComment": 60,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409594379,

          "createAt": 1762336405947

        }

      },

      "COL_CATEGORY_USER_ID": {

        "id": "COL_CATEGORY_USER_ID",

        "tableId": "TABLE_CATEGORY",

        "name": "user_id",

        "comment": "→ Users.user_id",

        "dataType": "BIGINT",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 2,

          "widthName": 60,

          "widthComment": 84,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762411088931,

          "createAt": 1762336406533

        }

      },

      "COL_CATEGORY_NAME": {

        "id": "COL_CATEGORY_NAME",

        "tableId": "TABLE_CATEGORY",

        "name": "name",

        "comment": "",

        "dataType": "VARCHAR(50)",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 75,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_CATEGORY_SLUG": {

        "id": "COL_CATEGORY_SLUG",

        "tableId": "TABLE_CATEGORY",

        "name": "slug",

        "comment": "",

        "dataType": "VARCHAR(50)",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 75,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_CATEGORY_VISIBILITY": {

        "id": "COL_CATEGORY_VISIBILITY",

        "tableId": "TABLE_CATEGORY",

        "name": "visibility",

        "comment": "",

        "dataType": "VARCHAR(20)",

        "default": "'PUBLIC'",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 75,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_CATEGORY_DISPLAY_ORDER": {

        "id": "COL_CATEGORY_DISPLAY_ORDER",

        "tableId": "TABLE_CATEGORY",

        "name": "display_order",

        "comment": "",

        "dataType": "INT",

        "default": "0",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 73,

          "widthComment": 60,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_CATEGORY_CREATED_AT": {

        "id": "COL_CATEGORY_CREATED_AT",

        "tableId": "TABLE_CATEGORY",

        "name": "created_at",

        "comment": "",

        "dataType": "TIMESTAMP",

        "default": "CURRENT_TIMESTAMP",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 65,

          "widthDefault": 122

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_CATEGORY_UPDATED_AT": {

        "id": "COL_CATEGORY_UPDATED_AT",

        "tableId": "TABLE_CATEGORY",

        "name": "updated_at",

        "comment": "",

        "dataType": "TIMESTAMP",

        "default": "CURRENT_TIMESTAMP",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 62,

          "widthComment": 60,

          "widthDataType": 65,

          "widthDefault": 122

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_SERIES_ID": {

        "id": "COL_SERIES_ID",

        "tableId": "TABLE_SERIES",

        "name": "series_id",

        "comment": "",

        "dataType": "BIGINT",

        "default": "",

        "options": 11,

        "ui": {

          "keys": 1,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409594379,

          "createAt": 1762336405947

        }

      },

      "COL_SERIES_USER_ID": {

        "id": "COL_SERIES_USER_ID",

        "tableId": "TABLE_SERIES",

        "name": "user_id",

        "comment": "→ Users.user_id",

        "dataType": "BIGINT",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 2,

          "widthName": 60,

          "widthComment": 84,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_SERIES_NAME": {

        "id": "COL_SERIES_NAME",

        "tableId": "TABLE_SERIES",

        "name": "name",

        "comment": "",

        "dataType": "VARCHAR(100)",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 81,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_SERIES_DESCRIPTION": {

        "id": "COL_SERIES_DESCRIPTION",

        "tableId": "TABLE_SERIES",

        "name": "description",

        "comment": "",

        "dataType": "TEXT",

        "default": "",

        "options": 0,

        "ui": {

          "keys": 0,

          "widthName": 61,

          "widthComment": 60,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_SERIES_SLUG": {

        "id": "COL_SERIES_SLUG",

        "tableId": "TABLE_SERIES",

        "name": "slug",

        "comment": "",

        "dataType": "VARCHAR(100)",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 81,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_SERIES_THUMBNAIL_URL": {

        "id": "COL_SERIES_THUMBNAIL_URL",

        "tableId": "TABLE_SERIES",

        "name": "thumbnail_url",

        "comment": "",

        "dataType": "VARCHAR(500)",

        "default": "",

        "options": 0,

        "ui": {

          "keys": 0,

          "widthName": 75,

          "widthComment": 60,

          "widthDataType": 81,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_SERIES_VISIBILITY": {

        "id": "COL_SERIES_VISIBILITY",

        "tableId": "TABLE_SERIES",

        "name": "visibility",

        "comment": "",

        "dataType": "VARCHAR(20)",

        "default": "'PUBLIC'",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 75,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_SERIES_CREATED_AT": {

        "id": "COL_SERIES_CREATED_AT",

        "tableId": "TABLE_SERIES",

        "name": "created_at",

        "comment": "",

        "dataType": "TIMESTAMP",

        "default": "CURRENT_TIMESTAMP",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 65,

          "widthDefault": 122

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_SERIES_UPDATED_AT": {

        "id": "COL_SERIES_UPDATED_AT",

        "tableId": "TABLE_SERIES",

        "name": "updated_at",

        "comment": "",

        "dataType": "TIMESTAMP",

        "default": "CURRENT_TIMESTAMP",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 62,

          "widthComment": 60,

          "widthDataType": 65,

          "widthDefault": 122

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_TAG_ID": {

        "id": "COL_TAG_ID",

        "tableId": "TABLE_TAG",

        "name": "tag_id",

        "comment": "",

        "dataType": "BIGINT",

        "default": "",

        "options": 11,

        "ui": {

          "keys": 1,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409594379,

          "createAt": 1762336405947

        }

      },

      "COL_TAG_NAME": {

        "id": "COL_TAG_NAME",

        "tableId": "TABLE_TAG",

        "name": "name",

        "comment": "",

        "dataType": "VARCHAR(50)",

        "default": "",

        "options": 12,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 75,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_TAG_SLUG": {

        "id": "COL_TAG_SLUG",

        "tableId": "TABLE_TAG",

        "name": "slug",

        "comment": "",

        "dataType": "VARCHAR(50)",

        "default": "",

        "options": 12,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 75,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_TAG_CREATED_AT": {

        "id": "COL_TAG_CREATED_AT",

        "tableId": "TABLE_TAG",

        "name": "created_at",

        "comment": "",

        "dataType": "TIMESTAMP",

        "default": "CURRENT_TIMESTAMP",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 65,

          "widthDefault": 122

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_POSTTAG_ID": {

        "id": "COL_POSTTAG_ID",

        "tableId": "TABLE_POSTTAG",

        "name": "id",

        "comment": "",

        "dataType": "BIGINT",

        "default": "",

        "options": 11,

        "ui": {

          "keys": 1,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409594379,

          "createAt": 1762336405947

        }

      },

      "COL_POSTTAG_POST_ID": {

        "id": "COL_POSTTAG_POST_ID",

        "tableId": "TABLE_POSTTAG",

        "name": "post_id",

        "comment": "→ Post.post_id",

        "dataType": "BIGINT",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 2,

          "widthName": 60,

          "widthComment": 79,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_POSTTAG_TAG_ID": {

        "id": "COL_POSTTAG_TAG_ID",

        "tableId": "TABLE_POSTTAG",

        "name": "tag_id",

        "comment": "→ Tag.tag_id",

        "dataType": "BIGINT",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 2,

          "widthName": 60,

          "widthComment": 69,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_POSTTAG_CREATED_AT": {

        "id": "COL_POSTTAG_CREATED_AT",

        "tableId": "TABLE_POSTTAG",

        "name": "created_at",

        "comment": "",

        "dataType": "TIMESTAMP",

        "default": "CURRENT_TIMESTAMP",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 65,

          "widthDefault": 122

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_IMAGE_ID": {

        "id": "COL_IMAGE_ID",

        "tableId": "TABLE_IMAGE",

        "name": "image_id",

        "comment": "",

        "dataType": "BIGINT",

        "default": "",

        "options": 11,

        "ui": {

          "keys": 1,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409594379,

          "createAt": 1762336405947

        }

      },

      "COL_IMAGE_USER_ID": {

        "id": "COL_IMAGE_USER_ID",

        "tableId": "TABLE_IMAGE",

        "name": "user_id",

        "comment": "→ Users.user_id",

        "dataType": "BIGINT",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 2,

          "widthName": 60,

          "widthComment": 84,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_IMAGE_ORIGINAL_NAME": {

        "id": "COL_IMAGE_ORIGINAL_NAME",

        "tableId": "TABLE_IMAGE",

        "name": "original_name",

        "comment": "",

        "dataType": "VARCHAR(255)",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 76,

          "widthComment": 60,

          "widthDataType": 81,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_IMAGE_STORED_NAME": {

        "id": "COL_IMAGE_STORED_NAME",

        "tableId": "TABLE_IMAGE",

        "name": "stored_name",

        "comment": "",

        "dataType": "VARCHAR(255)",

        "default": "",

        "options": 12,

        "ui": {

          "keys": 0,

          "widthName": 70,

          "widthComment": 60,

          "widthDataType": 81,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_IMAGE_URL": {

        "id": "COL_IMAGE_URL",

        "tableId": "TABLE_IMAGE",

        "name": "url",

        "comment": "",

        "dataType": "VARCHAR(500)",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 81,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_IMAGE_SIZE": {

        "id": "COL_IMAGE_SIZE",

        "tableId": "TABLE_IMAGE",

        "name": "size",

        "comment": "",

        "dataType": "BIGINT",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 60,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_IMAGE_MIME_TYPE": {

        "id": "COL_IMAGE_MIME_TYPE",

        "tableId": "TABLE_IMAGE",

        "name": "mime_type",

        "comment": "",

        "dataType": "VARCHAR(50)",

        "default": "",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 75,

          "widthDefault": 60

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      },

      "COL_IMAGE_CREATED_AT": {

        "id": "COL_IMAGE_CREATED_AT",

        "tableId": "TABLE_IMAGE",

        "name": "created_at",

        "comment": "",

        "dataType": "TIMESTAMP",

        "default": "CURRENT_TIMESTAMP",

        "options": 8,

        "ui": {

          "keys": 0,

          "widthName": 60,

          "widthComment": 60,

          "widthDataType": 65,

          "widthDefault": 122

        },

        "meta": {

          "updateAt": 1762409559342,

          "createAt": 1762336406533

        }

      }

    },

    "relationshipEntities": {

      "REL_USER_POST": {

        "id": "REL_USER_POST",

        "identification": false,

        "relationshipType": 16,

        "startRelationshipType": 2,

        "start": {

          "tableId": "TABLE_USERS",

          "columnIds": [

            "COL_USERS_ID"

          ],

          "x": 942.1053,

          "y": 249.0001,

          "direction": 8

        },

        "end": {

          "tableId": "TABLE_POST",

          "columnIds": [

            "COL_POST_USER_ID"

          ],

          "x": 941.2617,

          "y": 457.5883,

          "direction": 4

        },

        "meta": {

          "updateAt": 1762409591703,

          "createAt": 1762336073899

        }

      },

      "REL_USER_CATEGORY": {

        "id": "REL_USER_CATEGORY",

        "identification": false,

        "relationshipType": 16,

        "startRelationshipType": 2,

        "start": {

          "tableId": "TABLE_USERS",

          "columnIds": [

            "COL_USERS_ID"

          ],

          "x": 717.1053,

          "y": 137.0001,

          "direction": 1

        },

        "end": {

          "tableId": "TABLE_CATEGORY",

          "columnIds": [

            "COL_CATEGORY_USER_ID"

          ],

          "x": 589.5263,

          "y": 306.7368,

          "direction": 2

        },

        "meta": {

          "updateAt": 1762409591703,

          "createAt": 1762336073899

        }

      },

      "REL_USER_SERIES": {

        "id": "REL_USER_SERIES",

        "identification": false,

        "relationshipType": 16,

        "startRelationshipType": 2,

        "start": {

          "tableId": "TABLE_USERS",

          "columnIds": [

            "COL_USERS_ID"

          ],

          "x": 1167.1053000000002,

          "y": 193.0001,

          "direction": 2

        },

        "end": {

          "tableId": "TABLE_SERIES",

          "columnIds": [

            "COL_SERIES_USER_ID"

          ],

          "x": 1522.3684,

          "y": 450.8949,

          "direction": 1

        },

        "meta": {

          "updateAt": 1762409591703,

          "createAt": 1762336073899

        }

      },

      "REL_USER_IMAGE": {

        "id": "REL_USER_IMAGE",

        "identification": false,

        "relationshipType": 16,

        "startRelationshipType": 2,

        "start": {

          "tableId": "TABLE_USERS",

          "columnIds": [

            "COL_USERS_ID"

          ],

          "x": 1167.1053000000002,

          "y": 81.0001,

          "direction": 2

        },

        "end": {

          "tableId": "TABLE_IMAGE",

          "columnIds": [

            "COL_IMAGE_USER_ID"

          ],

          "x": 1518.4212,

          "y": 193.73680000000002,

          "direction": 1

        },

        "meta": {

          "updateAt": 1762409591703,

          "createAt": 1762336073899

        }

      },

      "REL_CATEGORY_POST": {

        "id": "REL_CATEGORY_POST",

        "identification": false,

        "relationshipType": 16,

        "startRelationshipType": 1,

        "start": {

          "tableId": "TABLE_CATEGORY",

          "columnIds": [

            "COL_CATEGORY_ID"

          ],

          "x": 589.5263,

          "y": 430.7368,

          "direction": 2

        },

        "end": {

          "tableId": "TABLE_POST",

          "columnIds": [

            "COL_POST_CATEGORY_ID"

          ],

          "x": 676.2617,

          "y": 665.5883,

          "direction": 1

        },

        "meta": {

          "updateAt": 1762409591703,

          "createAt": 1762336073899

        }

      },

      "REL_SERIES_POST": {

        "id": "REL_SERIES_POST",

        "identification": false,

        "relationshipType": 16,

        "startRelationshipType": 1,

        "start": {

          "tableId": "TABLE_SERIES",

          "columnIds": [

            "COL_SERIES_ID"

          ],

          "x": 1522.3684,

          "y": 586.8949,

          "direction": 1

        },

        "end": {

          "tableId": "TABLE_POST",

          "columnIds": [

            "COL_POST_SERIES_ID"

          ],

          "x": 1206.2617,

          "y": 561.5883,

          "direction": 2

        },

        "meta": {

          "updateAt": 1762409591703,

          "createAt": 1762336073899

        }

      },

      "REL_POST_POSTTAG": {

        "id": "REL_POST_POSTTAG",

        "identification": false,

        "relationshipType": 16,

        "startRelationshipType": 2,

        "start": {

          "tableId": "TABLE_POST",

          "columnIds": [

            "COL_POST_ID"

          ],

          "x": 1206.2617,

          "y": 769.5883,

          "direction": 2

        },

        "end": {

          "tableId": "TABLE_POSTTAG",

          "columnIds": [

            "COL_POSTTAG_POST_ID"

          ],

          "x": 1534.2105,

          "y": 781.2632,

          "direction": 1

        },

        "meta": {

          "updateAt": 1762409591703,

          "createAt": 1762336073899

        }

      },

      "REL_TAG_POSTTAG": {

        "id": "REL_TAG_POSTTAG",

        "identification": false,

        "relationshipType": 16,

        "startRelationshipType": 2,

        "start": {

          "tableId": "TABLE_TAG",

          "columnIds": [

            "COL_TAG_ID"

          ],

          "x": 1759.158,

          "y": 1002.6317,

          "direction": 4

        },

        "end": {

          "tableId": "TABLE_POSTTAG",

          "columnIds": [

            "COL_POSTTAG_TAG_ID"

          ],

          "x": 1759.7105,

          "y": 857.2632,

          "direction": 8

        },

        "meta": {

          "updateAt": 1762409591703,

          "createAt": 1762336073899

        }

      }

    },

    "indexEntities": {

      "md-zALLxrAbndE3boM5TC": {

        "id": "md-zALLxrAbndE3boM5TC",

        "name": "",

        "tableId": "TABLE_CATEGORY",

        "indexColumnIds": [],

        "seqIndexColumnIds": [],

        "unique": false,

        "meta": {

          "updateAt": 1762411107377,

          "createAt": 1762411107377

        }

      }

    },

    "indexColumnEntities": {},

    "memoEntities": {}

  }
}
```


## 프로젝트 구조

```
src/main/java/com/example/astarchia/
│
├── domain/                    # 도메인별로 구분 (추천!)
│   ├── user/
│   │   ├── entity/
│   │   │   └── User.java
│   │   ├── dto/
│   │   │   ├── request/
│   │   │   │   ├── UserCreateRequestDTO.java
│   │   │   │   ├── UserLoginRequestDTO.java
│   │   │   │   └── UserUpdateRequestDTO.java
│   │   │   └── response/
│   │   │       └── UserResponseDTO.java
│   │   ├── repository/
│   │   │   └── UserRepository.java
│   │   ├── service/
│   │   │   └── UserService.java
│   │   └── controller/
│   │       └── UserController.java
│   │
│   ├── post/
│   │   ├── entity/
│   │   │   └── Post.java
│   │   ├── dto/
│   │   ├── repository/
│   │   ├── service/
│   │   └── controller/
│   │
│   ├── category/
│   ├── tag/
│   ├── series/
│   └── comment/
│
├── global/                    # 공통 기능
│   ├── config/               # 설정
│   │   ├── SecurityConfig.java
│   │   └── JpaConfig.java
│   ├── exception/            # 예외 처리
│   │   ├── CustomException.java
│   │   └── GlobalExceptionHandler.java
│   └── util/                 # 유틸리티
│       └── JwtUtil.java
│
└── AstarchiaApplication.java
````

프로젝트 생성

1. Entity 설계 
2. Repository 생성 
3. DTO 설계 (Request/Response 분리) 
4. Service 구현 (비즈니스 로직) 
5. Controller 구현 (API 엔드포인트) 
6. 테스트
