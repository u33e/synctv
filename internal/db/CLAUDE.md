# [根目录](../CLAUDE.md) > internal/db (数据库访问层)

> [!NOTE]
> 面包屑: [SyncTV 项目根](../CLAUDE.md) / internal / db (数据库访问层)

## 模块职责

`internal/db` 模块提供数据库访问的抽象层，封装 GORM 操作，提供常用查询方法和事务支持。作为数据模型层 (`internal/model`) 和业务逻辑层 (`internal/op`) 之间的桥梁。

## 目录结构

```
internal/db/
├── db.go              # 数据库初始化和通用方法
├── update.go          # 数据库迁移
├── user.go            # 用户相关操作
├── room.go            # 房间相关操作
├── movie.go           # 影片相关操作
├── member.go          # 成员相关操作
├── setting.go         # 设置相关操作
├── vendorBackend.go   # 厂商后端操作
└── vendorRecord.go    # 厂商记录操作
```

## 入口与启动

### 初始化

```go
func Init(d *gorm.DB, t conf.DatabaseType) error
```

**初始化流程**:
1. 保存数据库连接
2. 执行数据库迁移
3. 初始化 Guest 用户
4. 初始化 Root 用户

### 默认用户

**Root 用户**:
- 用户名: `root`
- 密码: `root`
- 角色: `RoleRoot`
- 首次启动自动创建

**Guest 用户**:
- 用户名: `guest`
- ID: 固定 `00000000000000000000000000000001`
- 角色: `RoleUser`
- 用于未登录用户访问

## 核心功能

### 通用查询工具

**分页查询**:
```go
func Paginate(page, pageSize int) func(db *gorm.DB) *gorm.DB
```

**排序**:
```go
func OrderByAsc(column string) func(db *gorm.DB) *gorm.DB
func OrderByDesc(column string) func(db *gorm.DB) *gorm.DB
func OrderByCreatedAtAsc(db *gorm.DB) *gorm.DB
func OrderByCreatedAtDesc(db *gorm.DB) *gorm.DB
```

**预加载**:
```go
func WithUser(db *gorm.DB) *gorm.DB
func PreloadRoomMembers(scopes ...func(*gorm.DB) *gorm.DB) func(db *gorm.DB) *gorm.DB
func PreloadUserProviders(scopes ...func(*gorm.DB) *gorm.DB) func(db *gorm.DB) *gorm.DB
```

**条件查询**:
```go
func WhereRoomID(roomID string) func(db *gorm.DB) *gorm.DB
func WhereUserID(userID string) func(db *gorm.DB) *gorm.DB
func WhereCreatorID(creatorID string) func(db *gorm.DB) *gorm.DB
func WhereLike(column, value string) func(db *gorm.DB) *gorm.DB
```

**复杂查询**:
```go
func WhereRoomNameLikeOrCreatorInOrIDLike(name string, ids []string, id string) func(db *gorm.DB) *gorm.DB
func WhereMovieNameLikeOrURLLike(name, url string) func(db *gorm.DB) *gorm.DB
func WhereUsernameLikeOrIDIn(name string, ids []string) func(db *gorm.DB) *gorm.DB
```

### 事务支持

```go
func Transactional(txFunc func(*gorm.DB) error) (err error)
```

**使用示例**:
```go
err := db.Transactional(func(tx *gorm.DB) error {
    if err := tx.Create(&user).Error; err != nil {
        return err
    }
    if err := tx.Create(&room).Error; err != nil {
        return err
    }
    return nil
})
```

### 错误处理

```go
type NotFoundError string

func (e NotFoundError) Error() string

func HandleNotFound(err error, errMsg ...string) error
func HandleUpdateResult(result *gorm.DB, entityName string) error
```

## 对外接口

### 用户操作

```go
func CreateUser(username, password string, opts ...UserOption) (*model.User, error)
func GetUserByID(id string, opts ...UserQueryOption) (*model.User, error)
func GetUserByUsername(username string, opts ...UserQueryOption) (*model.User, error)
func UpdateUser(userID string, opts ...UserUpdateOption) error
func DeleteUser(id string) error
func BanUser(id string) error
func UnbanUser(id string) error
func SearchUsers(keyword string, page, pageSize int) ([]*model.User, int64, error)
```

### 房间操作

```go
func CreateRoom(name string, creatorID string, opts ...RoomOption) (*model.Room, error)
func GetRoomByID(id string, opts ...RoomQueryOption) (*model.Room, error)
func GetRoomByName(name string) (*model.Room, error)
func UpdateRoom(roomID string, opts ...RoomUpdateOption) error
func DeleteRoom(id string) error
func GetRooms(page, pageSize int, opts ...RoomQueryOption) ([]*model.Room, int64, error)
```

### 影片操作

```go
func CreateMovie(roomID, creatorID string, base *model.MovieBase, position uint) (*model.Movie, error)
func GetMovieByID(id string) (*model.Movie, error)
func GetMoviesByRoomID(roomID string) ([]*model.Movie, error)
func UpdateMovie(id string, opts ...MovieUpdateOption) error
func DeleteMovie(id string) error
```

### 成员操作

```go
func CreateRoomMember(roomID, userID string, role model.RoomMemberRole) (*model.RoomMember, error)
func GetRoomMember(roomID, userID string) (*model.RoomMember, error)
func UpdateRoomMemberRole(roomID, userID string, role model.RoomMemberRole) error
func UpdateRoomMemberStatus(roomID, userID string, status model.RoomMemberStatus) error
func DeleteRoomMember(roomID, userID string) error
func GetRoomMembers(roomID string, page, pageSize int) ([]*model.RoomMember, int64, error)
```

### 设置操作

```go
func GetSetting(name string) (*model.Setting, error)
func SetSetting(name, value string) error
func DeleteSetting(name string) error
func GetSettings(group model.SettingGroup) ([]*model.Setting, error)
```

### 厂商后端操作

```go
func CreateVendorBackend(backend *model.VendorBackend) error
func GetVendorBackend(endpoint string) (*model.VendorBackend, error)
func GetAllVendorBackend() ([]*model.VendorBackend, error)
func SaveVendorBackend(backend *model.VendorBackend) error
func DeleteVendorBackend(endpoint string) error
func EnableVendorBackend(endpoint string) error
func DisableVendorBackend(endpoint string) error
```

## 关键依赖

| 依赖 | 用途 |
|------|------|
| `gorm.io/gorm` | ORM 框架 |
| `github.com/synctv-org/synctv/internal/model` | 数据模型 |
| `github.com/synctv-org/synctv/internal/conf` | 配置类型 |

## 数据库支持

支持以下数据库类型:

| 类型 | 说明 |
|------|------|
| `sqlite3` | SQLite (默认) |
| `mysql` | MySQL/MariaDB |
| `postgres` | PostgreSQL |

根据数据库类型自动调整 SQL 语法 (如 LIKE/ILIKE)。

## 常见问题 (FAQ)

### 如何处理数据库升级?

```go
func UpgradeDatabase() error
```

该函数会自动执行所有迁移操作。

### 如何处理"未找到"错误?

使用 `HandleNotFound`:

```go
user, err := db.GetUserByID(id)
err = db.HandleNotFound(err, "user")
if errors.As(err, &db.NotFoundError{}) {
    // 处理未找到情况
}
```

### 如何执行复杂查询?

使用 Scope 组合:

```go
db.DB().Scopes(
    db.WhereRoomID(roomID),
    db.OrderByNameAsc(),
    db.Paginate(1, 10),
).Find(&movies)
```

## 相关文件清单

| 文件路径 | 说明 |
|----------|------|
| [db.go](db.go) | 数据库初始化和通用方法 |
| [update.go](update.go) | 数据库迁移 |
| [user.go](user.go) | 用户操作 |
| [room.go](room.go) | 房间操作 |
| [movie.go](movie.go) | 影片操作 |
| [member.go](member.go) | 成员操作 |
| [setting.go](setting.go) | 设置操作 |
| [vendorBackend.go](vendorBackend.go) | 厂商后端操作 |
| [vendorRecord.go](vendorRecord.go) | 厂商记录操作 |

## 变更记录 (Changelog)

| 日期 | 变更内容 |
|------|----------|
| 2026-01-10T00:45:00+0800 | 创建文档 |
