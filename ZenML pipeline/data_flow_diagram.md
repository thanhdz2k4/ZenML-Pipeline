# SƠ ĐỒ LUỒNG DỮ LIỆU - ZENML PIPELINE

## 🔄 TỔNG QUAN LUỒNG CHÍNH

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   INPUT DATA    │───▶│  ZENML PIPELINE  │───▶│  OUTPUT DATA    │
└─────────────────┘    └──────────────────┘    └─────────────────┘
│                      │                      │
├─ user_full_name     ├─ get_or_create_user ├─ invocation_id
└─ links[]            └─ crawl_links        └─ crawled_data
```

## 📊 CHI TIẾT TỪNG BƯỚC

### BƯỚC 1: PIPELINE INITIALIZATION
```
digital_data_etl(user_full_name: str, links: list[str])
                    │
                    ▼
            ┌───────────────┐
            │ ZenML Context │
            │ - Metadata    │
            │ - Logging     │
            │ - Tracking    │
            └───────────────┘
```

### BƯỚC 2: USER MANAGEMENT
```
get_or_create_user(user_full_name)
            │
            ▼
    ┌─────────────────┐
    │ utils.split_    │
    │ user_full_name  │
    └─────────────────┘
            │
            ▼
    ┌─────────────────┐
    │ first_name      │
    │ last_name       │
    └─────────────────┘
            │
            ▼
    ┌─────────────────┐
    │ UserDocument.   │
    │ get_or_create() │
    └─────────────────┘
            │
            ▼
    ┌─────────────────┐
    │ MongoDB Query   │
    │ Collection:     │
    │ "users"         │
    └─────────────────┘
            │
            ▼
    ┌─────────────────┐    ┌─────────────────┐
    │ User EXISTS?    │NO  │ Create New User │
    │                 │───▶│ & Save to DB    │
    └─────────────────┘    └─────────────────┘
            │YES                   │
            ▼                      ▼
    ┌─────────────────┐    ┌─────────────────┐
    │ Return Existing │◀───│ Return New User │
    │ UserDocument    │    │ Document        │
    └─────────────────┘    └─────────────────┘
            │                      │
            └──────────┬───────────┘
                       ▼
            ┌─────────────────┐
            │ UserDocument    │
            │ - id: UUID4     │
            │ - first_name    │
            │ - last_name     │
            └─────────────────┘
```

### BƯỚC 3: CRAWLER INITIALIZATION
```
crawl_links(user: UserDocument, links: list[str])
                    │
                    ▼
        ┌─────────────────────┐
        │ CrawlerDispatcher   │
        │ .build()            │
        └─────────────────────┘
                    │
                    ▼
        ┌─────────────────────┐
        │ Register Crawlers:  │
        │ ├─ LinkedIn         │
        │ ├─ Medium           │
        │ └─ GitHub           │
        └─────────────────────┘
                    │
                    ▼
        ┌─────────────────────┐
        │ Pattern Registry:   │
        │ ├─ linkedin.com     │
        │ ├─ medium.com       │
        │ ├─ github.com       │
        │ └─ fallback         │
        └─────────────────────┘
```

### BƯỚC 4: LINK PROCESSING LOOP
```
for link in tqdm(links):
        │
        ▼
┌─────────────────┐
│ _crawl_link()   │
│ (dispatcher,    │
│  link, user)    │
└─────────────────┘
        │
        ▼
┌─────────────────┐
│ URL Analysis    │
│ urlparse(link)  │
│ ├─ domain       │
│ └─ netloc       │
└─────────────────┘
        │
        ▼
┌─────────────────┐
│ Crawler         │
│ Selection       │
│ (Pattern Match) │
└─────────────────┘
        │
        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ LinkedIn        │    │ Medium          │    │ GitHub          │    │ Custom Article  │
│ Crawler         │    │ Crawler         │    │ Crawler         │    │ Crawler         │
│ (if match)      │    │ (if match)      │    │ (if match)      │    │ (fallback)      │
└─────────────────┘    └─────────────────┘    └─────────────────┘    └─────────────────┘
        │                       │                       │                       │
        └───────────────────────┼───────────────────────┼───────────────────────┘
                                ▼
                    ┌─────────────────┐
                    │ crawler.extract │
                    │ (link, user)    │
                    └─────────────────┘
                                │
                                ▼
                    ┌─────────────────┐
                    │ Data Extraction │
                    │ & Processing    │
                    └─────────────────┘
                                │
                                ▼
                    ┌─────────────────┐
                    │ Document        │
                    │ Creation        │
                    └─────────────────┘
                                │
                                ▼
                    ┌─────────────────┐
                    │ Save to MongoDB │
                    │ Collections:    │
                    │ ├─ posts        │
                    │ ├─ articles     │
                    │ └─ repositories │
                    └─────────────────┘
                                │
                                ▼
                    ┌─────────────────┐
                    │ Return Status:  │
                    │ (success, domain)│
                    └─────────────────┘
```

### BƯỚC 5: METADATA COLLECTION
```
_add_to_metadata(metadata, domain, success)
                    │
                    ▼
        ┌─────────────────┐
        │ Per-Domain      │
        │ Statistics:     │
        │ ├─ successful   │
        │ └─ total        │
        └─────────────────┘
                    │
                    ▼
        ┌─────────────────┐
        │ Aggregate       │
        │ Metadata        │
        └─────────────────┘
```

## 🏗️ KIẾN TRÚC DỮ LIỆU

### DATABASE SCHEMA
```
MongoDB Database: "twin"
│
├─ Collection: "users"
│  └─ UserDocument
│     ├─ _id: string (UUID)
│     ├─ first_name: string
│     └─ last_name: string
│
├─ Collection: "posts" 
│  └─ PostDocument
│     ├─ _id: string (UUID)
│     ├─ content: dict
│     ├─ platform: string
│     ├─ author_id: string (UUID)
│     ├─ author_full_name: string
│     ├─ image: string (optional)
│     └─ link: string (optional)
│
├─ Collection: "articles"
│  └─ ArticleDocument
│     ├─ _id: string (UUID)
│     ├─ content: dict
│     ├─ platform: string
│     ├─ author_id: string (UUID)
│     ├─ author_full_name: string
│     └─ link: string
│
└─ Collection: "repositories"
   └─ RepositoryDocument
      ├─ _id: string (UUID)
      ├─ content: dict
      ├─ platform: string
      ├─ author_id: string (UUID)
      ├─ author_full_name: string
      ├─ name: string
      └─ link: string
```

## 📈 LUỒNG DỮ LIỆU DETAIL

### INPUT ➜ PROCESSING ➜ OUTPUT
```
INPUT DATA:
┌─────────────────────────────────┐
│ user_full_name: "Nguyen Van A"  │
│ links: [                        │
│   "https://linkedin.com/in/...", │
│   "https://medium.com/@...",     │
│   "https://github.com/..."      │
│ ]                               │
└─────────────────────────────────┘
                │
                ▼
PROCESSING LAYER:
┌─────────────────────────────────┐
│ ZenML Pipeline Execution        │
│ ├─ Step 1: User Management     │
│ │  └─ MongoDB: users collection│
│ └─ Step 2: Data Crawling       │
│    ├─ Pattern Matching         │
│    ├─ Data Extraction          │
│    └─ MongoDB: content collections│
└─────────────────────────────────┘
                │
                ▼
OUTPUT DATA:
┌─────────────────────────────────┐
│ invocation_id: "step_12345..."  │
│                                 │
│ Side Effects:                   │
│ ├─ User stored in MongoDB       │
│ ├─ Content stored in MongoDB    │
│ ├─ ZenML metadata logged        │
│ └─ Execution logs generated     │
└─────────────────────────────────┘
```

## 🔍 ERROR HANDLING & MONITORING

### ERROR FLOW
```
Exception in crawler.extract()
            │
            ▼
┌─────────────────┐
│ Log Error       │
│ (loguru)        │
└─────────────────┘
            │
            ▼
┌─────────────────┐
│ Return          │
│ (False, domain) │
└─────────────────┘
            │
            ▼
┌─────────────────┐
│ Continue with   │
│ Next Link       │
└─────────────────┘
```

### MONITORING FLOW
```
ZenML Step Context
            │
            ▼
┌─────────────────┐
│ Metadata        │
│ Collection      │
└─────────────────┘
            │
            ▼
┌─────────────────┐
│ Progress        │
│ Tracking        │
│ (tqdm)          │
└─────────────────┘
            │
            ▼
┌─────────────────┐
│ Success Rate    │
│ Calculation     │
└─────────────────┘
```

## 🚀 EXTENSION POINTS

### Thêm Crawler Mới
```
New Platform (e.g., Twitter)
            │
            ▼
┌─────────────────┐
│ Implement       │
│ BaseCrawler     │
└─────────────────┘
            │
            ▼
┌─────────────────┐
│ Register in     │
│ Dispatcher      │
└─────────────────┘
            │
            ▼
┌─────────────────┐
│ Auto-routing    │
│ Available       │
└─────────────────┘
```

### Thêm Document Type
```
New Content Type
            │
            ▼
┌─────────────────┐
│ Inherit from    │
│ Document        │
└─────────────────┘
            │
            ▼
┌─────────────────┐
│ Define Settings │
│ (collection)    │
└─────────────────┘
            │
            ▼
┌─────────────────┐
│ Auto-CRUD       │
│ Operations      │
└─────────────────┘
```

---

**KẾT LUẬN**: Đây là một kiến trúc data flow rất scalable và maintainable, sử dụng các pattern tốt để đảm bảo tính mở rộng và độ tin cậy cao! 🎯
