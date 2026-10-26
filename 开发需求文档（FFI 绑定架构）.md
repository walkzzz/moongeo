# moon-geo 开发需求文档（FFI 绑定架构）

> **版本**：v3.0（FFI 重构版）
> **日期**：2026-10-26
> **替代**：v2.0《geo 统一空间数学引擎 · 开发立项技术规范》
> **核心转变**：从"纯 MoonBit 自研一切"转向"FFI 绑定成熟 C 库 + MoonBit 薄封装 + 纯 MoonBit 基础层"

---

## 0. 项目定位

### 0.1 一句话定位

**moon-geo 是 MoonBit 生态的几何计算库：以纯 MoonBit 实现基础几何类型与算法，通过 FFI 绑定 PROJ/GEOS/GDAL 获得工业级 CRS 投影、几何运算与格式 I/O 能力，编译为 WASM 供 Web 端使用。**

### 0.2 定位对比

| 维度 | v2.0（原立项） | v3.0（本文档） |
|------|--------------|--------------|
| 定位 | 统一空间数学引擎 / 范畴化范式定义者 | MoonBit 生态几何计算库 |
| 对标 | CGAL + GDAL + ArcGIS | Shapely 级可用性 + WASM 友好 |
| 时间线 | 20 年（2026→2046） | 2–3 年到 1.0 |
| 实现方式 | 纯 MoonBit 自研全部 | 纯 MoonBit 基础层 + FFI 绑定 C 库 |
| 承诺 | 核心架构终身稳定 | 1.0 后 API 稳定，0.x 阶段可变 |
| 数学定位 | 复射影代数几何 + 代数拓扑内核 | 实用计算几何（欧氏 2D/3D 为主） |
| 目标用户 | 全行业空间计算 | MoonBit 开发者 + Web WASM 地图/可视化 |

### 0.3 目标用户

| 用户 | 当前用什么 | 为什么用 moon-geo | 需要什么功能 |
|------|-----------|-----------------|------------|
| MoonBit 应用开发者 | 无（生态缺库） | MoonBit 原生，类型安全 | 基础几何 + 求交 + 变换 + I/O |
| Web 地图/可视化开发者 | Turf.js / geos-wasm | 纯 WASM、小体积、MoonBit 类型安全 | 几何运算 + CRS + GeoJSON |
| 嵌入式/边缘计算 | 自研 C 几何 | MoonBit 编译 native，无 GC 依赖可选 | 基础几何（core-only 构建） |

### 0.4 非目标（明确不做）

- ❌ 不做 CGAL 正面替代（CGAL 25 年积累，不可追）
- ❌ 不做范畴化数学范式定义（审美非需求）
- ❌ 不做形式化验证（归属在 Lean4/Coq）
- ❌ 不做 20 年路线图（不可承诺）
- ❌ 不做全行业领域适配层（CAD/GIS/仿真/机器人）
- ❌ 不自研 PROJ 投影公式（30 年积累，FFI 绑定）
- ❌ 不自研 Shapefile/GeoTIFF/LAS 格式解析（FFI 绑定 GDAL）

---

## 1. 架构总览

### 1.1 三层架构

```
┌─────────────────────────────────────────────────────────────┐
│                     api/  公共 API 层                         │
│   MoonBit 类型安全门面，统一入口，用户只接触这一层              │
│   geometry / crs / io / ops 四个模块                         │
└─────────────────────────────────────────────────────────────┘
                    │                    │
          ┌─────────┴──────┐    ┌───────┴────────┐
          ▼                ▼    ▼                ▼
┌─────────────────┐  ┌──────────────────────────────┐
│  core/  纯 MoonBit │  │  ffi/  C 库绑定（MoonBit FFI）  │
│  基础几何与算法    │  │  proj / geos / gdal / (blas)   │
│  无 C 依赖        │  │  RAII wrapper + 错误转换       │
└─────────────────┘  └──────────────────────────────┘
          │                        │
          │                        ▼
          │              ┌─────────────────┐
          │              │  C/C++ 库（外部）  │
          │              │  PROJ / GEOS /   │
          │              │  GDAL / BLAS     │
          │              └─────────────────┘
          ▼
┌─────────────────┐
│  WASM / Native   │
│  构建目标         │
└─────────────────┘
```

### 1.2 设计原则

1. **FFI 优先于自研**：凡有成熟 C 库且自研成本高者，FFI 绑定（PROJ、GEOS、GDAL）
2. **纯 MoonBit 保留基础**：几何类型、简单算法、数据结构用纯 MoonBit（无 C 依赖、可独立编译为最小 WASM）
3. **API 门面统一**：用户只接触 `api/` 层，不直接接触 `ffi/`，C 库细节被封装
4. **渐进式 C 依赖**：三种构建 profile，用户按需选择 C 依赖范围
5. **RAII 内存管理**：C 指针封装为 MoonBit 资源类型，自动释放，无手动 free
6. **错误边界转换**：C 库错误码转换为 MoonBit `raise GeoError`，不泄漏 C 细节

### 1.3 三种构建 Profile

| Profile | 包含 | C 依赖 | WASM 体积（估） | 适用场景 |
|---------|------|--------|--------------|---------|
| `core-only` | core/ + api/（geometry 部分） | 无 | ~100KB | 嵌入式、最小 WASM、教学 |
| `with-proj` | core-only + ffi/proj | PROJ | ~2MB | Web 地图、CRS 变换 |
| `full` | with-proj + ffi/geos + ffi/gdal | PROJ+GEOS+GDAL | ~10MB | 服务端 GIS、完整格式 I/O |

**用户按需选择**：只需基础几何的用 core-only，需要投影的加 PROJ，需要完整 GIS 的用 full。这比"一个巨大的 WASM 包含一切"实用得多。

---

## 2. 核心层 `core/`（纯 MoonBit，无 C 依赖）

### 2.1 职责

纯 MoonBit 实现的几何基础类型与算法。**无任何 C 依赖**，可独立编译为最小 WASM。这是项目的地基，也是 MoonBit 生态的纯原生贡献。

### 2.2 模块划分

```
core/
├── util/           # 数值基础（保留现有）
│   ├── num.mbt     # ApproxEq, NumExt, Field, epsilon 管理
│   ├── cpx.mbt     # 复数（保留，供远期射影运算）
│   ├── vec.mbt     # Vec2, Vec3
│   ├── mat.mbt     # Mat3, Mat4
│   └── quat.mbt    # 四元数
│
├── geom/           # 几何图元
│   ├── point.mbt   # Point2D, Point3D
│   ├── line.mbt    # Line2D, LineSegment2D, Ray2D
│   ├── plane.mbt   # Plane3D
│   ├── circle.mbt  # Circle2D
│   ├── polygon.mbt # Polygon2D（顶点环 + 内孔）
│   └── aabb.mbt    # AABB2D, AABB3D
│
├── intersect/      # 基础求交（纯 MoonBit）
│   ├── line_line.mbt
│   ├── line_circle.mbt
│   ├── circle_circle.mbt
│   ├── segment_segment.mbt
│   └── point_polygon.mbt
│
├── transform/      # 变换
│   ├── affine.mbt  # Affine2D, Affine3D（平移/旋转/缩放/复合）
│   └── projective.mbt  # 齐次坐标变换（保留，供射影运算）
│
├── bvh/            # 包围体层级树
│   └── bvh.mbt     # BVH 构建（中值分裂）+ 查询
│
├── hemesh/         # 半边数据结构
│   └── half_edge.mbt  # HEMesh + from_triangles + 邻接查询
│
├── delaunay/       # Delaunay 三角剖分
│   └── triangulate.mbt  # Bowyer-Watson + delaunay_to_hemesh
│
├── convex/         # 凸包
│   └── hull.mbt    # Andrew's monotone chain（2D）
│
└── clip/           # 多边形裁剪
    └── sutherland_hodgman.mbt  # 凸裁剪窗口
```

### 2.3 与现有代码的映射

| 现有路径 | 迁移到 | 说明 |
|---------|-------|------|
| `util/` | `core/util/` | 原样保留 |
| `alg/geo.mbt`, `alg/point.mbt`, `alg/line.mbt` | `core/geom/` | 拆分到图元文件 |
| `alg/intersect.mbt` | `core/intersect/` | 拆分到独立文件 |
| `alg/xform.mbt` | `core/transform/` | 重命名 |
| `bv/` | `core/bvh/` | 原样保留 |
| `brep/half_edge.mbt` | `core/hemesh/` | 重命名 |
| `algo/triangulate.mbt` | `core/delaunay/` | 原样保留 |
| `algo/convex_hull.mbt` | `core/convex/` | 原样保留 |
| `algo/clip.mbt`（如有） | `core/clip/` | — |

### 2.4 删除的层

| 现有层 | 处理 | 理由 |
|-------|------|------|
| `atop/`（拓扑空间/流形/同调） | **删除** | 无实质实现，不被下层依赖，范畴抽象非目标 |
| `space/`（Euclid2/3, Proj3, Spherical3 等） | **降级为 core/geom 中的类型** | 空间类型不需要独立层，几何类型自带空间语义 |
| `algo/boolean.mbt`, `algo/buffer.mbt` | **移至 ffi/geos** | 布尔运算/缓冲区由 GEOS 提供 |
| `algo/topo_relate.mbt`（DE-9IM） | **移至 ffi/geos** | DE-9IM 由 GEOS 提供 |
| `parallel/`, `domain/`, `eco/`, `draw/` | **删除** | 远期非目标 |

### 2.5 core 层的边界

core 层**只做**：
- 几何类型的构造与属性查询（面积、周长、质心、AABB）
- 基础求交（线-线、线-圆、圆-圆、点-多边形）
- 变换（仿射、齐次）
- 空间索引（BVH）
- 拓扑数据结构（半边）
- 三角剖分（Delaunay）
- 凸包
- 多边形裁剪（凸窗口）

core 层**不做**：
- 任意多边形布尔运算（→ GEOS）
- 缓冲区/偏移曲线（→ GEOS）
- DE-9IM 空间关系（→ GEOS）
- CRS 投影变换（→ PROJ）
- 格式 I/O（GeoJSON 除外，→ core；其余 → GDAL）
- 精确谓词（exact predicates，→ GEOS/CGAL 远期）

---

## 3. FFI 绑定层 `ffi/`

### 3.1 总体设计

每个 FFI 绑定是一个 MoonBit 包，包含：
- **C 头文件声明**：`extern "C"` 函数签名
- **RAII Wrapper**：C 指针封装为 MoonBit struct，`Drop` trait 自动释放
- **错误转换**：C 返回码 → `raise GeoError`
- **类型映射**：C 类型 ↔ MoonBit 类型（如 `OGRGeometry*` ↔ `api/Geometry`）

### 3.2 ffi/proj — PROJ 绑定

**绑定库**：[PROJ](https://proj.org/)（OSGeo，30+ 年，EPSG 全量支持）

**替换的现有代码**：`platform/crs/projection.mbt`（手写 UTM/Mercator）

**能力对比**：

| 能力 | 现有（手写） | PROJ FFI |
|------|------------|----------|
| UTM 正反算 | ✅ | ✅ |
| Web Mercator | ✅ | ✅ |
| EPSG 支持 | ❌（仅硬编码 WGS84/CGCS2000） | ✅ 全量 EPSG 数据库 |
| Lambert / Albers / Stereographic 等 | ❌ | ✅ 100+ 投影 |
| Datum 变换（7 参数、网格偏移） | ❌ | ✅ |
| 时间相关坐标变换（动坐标） | ❌ | ✅ |
| 精度验证 | ❌ | ✅ OGC 测试套件 |

**API 设计**：

```moonbit
// ffi/proj/proj.mbt（概念设计）

// RAII wrapper：C 的 PJ_CONTEXT* 封装
pub struct ProjContext {
  ctx : RawPointer  // PJ_CONTEXT*
}

impl Drop for ProjContext {
  fn drop(self) {
    proj_context_destroy(self.ctx)
  }
}

// RAII wrapper：C 的 PJ* 坐标变换封装
pub struct ProjTransform {
  pj : RawPointer  // PJ*
  context : ProjContext  // 持有 context 引用
}

impl Drop for ProjTransform {
  fn drop(self) {
    proj_destroy(self.pj)
  }
}

/// 从 EPSG 编码创建坐标变换
pub fn ProjTransform::from_epsg(
  ctx : ProjContext,
  source_epsg : Int,
  target_epsg : Int,
) -> ProjTransform raise GeoError {
  let pj = proj_create_crs_to_crs(
    ctx.ctx,
    "EPSG:\{source_epsg}",
    "EPSG:\{target_epsg}",
    RawPointer::null(),
  )
  if pj == RawPointer::null() {
    raise GeoError::FFIError("PROJ: failed to create transform")
  }
  { pj, context: ctx, }
}

/// 执行坐标变换（批量）
pub fn ProjTransform::transform(
  self : ProjTransform,
  coords : Array[(Double, Double)],
) -> Array[(Double, Double)] raise GeoError {
  // 调用 proj_trans_array，转换错误码
  ...
}
```

**迁移路径**：
1. 现有 `platform/crs/crs.mbt` 的 `Ellipsoid`/`Datum`/`CrsKind` 保留为 MoonBit 类型（用户接口）
2. `platform/crs/projection.mbt` 的 `UtmProjection::forward/inverse` 改为调用 `ffi/proj`
3. 新增 `api/crs.mbt` 提供 `transform(epsg_from, epsg_to, coords)` 统一入口
4. 现有 UTM 手写实现保留为 `core/` 的 fallback（PROJ 不可用时降级）

### 3.3 ffi/geos — GEOS 绑定

**绑定库**：[GEOS](https://libgeos.org/)（OSGeo，20+ 年，PostGIS 底层）

**替换的现有代码**：`algo/boolean.mbt`（计划但未实现）、`algo/buffer.mbt`（计划但未实现）

**能力**：

| 能力 | GEOS 提供 | core 层是否覆盖 |
|------|----------|--------------|
| 几何布尔运算（union/intersection/difference/symmetricDifference） | ✅ | ❌（仅凸裁剪） |
| 缓冲区（buffer，含圆角/平角/miter） | ✅ | ❌ |
| DE-9IM 空间关系（contains/within/intersects/touches/crosses/overlaps/disjoint） | ✅ | ❌ |
| 几何简化（Douglas-Peucker / TopologyPreserving） | ✅ | ❌ |
| 凸包（任意维度） | ✅ | ✅（2D 已有，3D 可用 GEOS） |
| Delaunay 三角剖分 | ✅ | ✅（已有，但 GEOS 更鲁棒） |
| WKT / WKB 解析与序列化 | ✅ | ❌ |
| 最近点 / 距离 | ✅ | 部分（core 有点-多边形距离） |
| 线合并 / 线分割 / 节点化 | ✅ | ❌ |

**API 设计**：

```moonbit
// ffi/geos/geos.mbt（概念设计）

// RAII wrapper：GEOSGeometry*
pub struct GeosGeometry {
  geom : RawPointer  // GEOSGeometry*
  ctx : GeosContext   // 持有 context
}

impl Drop for GeosGeometry {
  fn drop(self) {
    GEOSGeom_destroy_r(self.ctx.handle, self.geom)
  }
}

/// 从 MoonBit Polygon 创建 GEOS 几何
pub fn GeosGeometry::from_polygon(poly : @core.geom.Polygon2D) -> GeosGeometry raise GeoError {
  // 构造 GEOSCoordinateSequence → GEOSLinearRing → GEOSPolygon
  ...
}

/// 布尔交集
pub fn intersection(a : GeosGeometry, b : GeosGeometry) -> GeosGeometry raise GeoError {
  let result = GEOSIntersection_r(a.ctx.handle, a.geom, b.geom)
  if result == RawPointer::null() {
    raise GeoError::FFIError("GEOS: intersection failed")
  }
  { geom: result, ctx: a.ctx, }
}

/// 转回 MoonBit Polygon
pub fn GeosGeometry::to_polygon(self : GeosGeometry) -> @core.geom.Polygon2D raise GeoError {
  ...
}
```

### 3.4 ffi/gdal — GDAL 绑定

**绑定库**：[GDAL](https://gdal.org/)（OSGeo，25+ 年，地理 I/O 事实标准）

**替换的现有代码**：`platform/io/` 的格式读写（GeoJSON 保留纯 MoonBit，其余 FFI）

**能力**：

| 格式 | 现有（手写） | GDAL FFI |
|------|------------|----------|
| GeoJSON | ✅ 纯 MoonBit | 保留纯 MoonBit（简单格式，无 C 依赖） |
| STL | ✅ 纯 MoonBit | 保留纯 MoonBit（简单格式） |
| Shapefile | ❌ | ✅ GDAL OGR |
| GeoTIFF | ❌ | ✅ GDAL Raster |
| LAS/LAZ（点云） | ❌ | ✅ PDAL 或 GDAL |
| DXF | ❌ | ✅ GDAL OGR |
| KML | ❌ | ✅ GDAL OGR |
| GML | ❌ | ✅ GDAL OGR |
| CSV（带坐标） | ❌ | ✅ GDAL OGR |
| PostGIS 读写 | ❌ | ✅ GDAL OGR |

**设计原则**：GeoJSON 和 STL 这类简单格式保留纯 MoonBit 实现（已在 core 中，无 C 依赖）。复杂格式（Shapefile 的 shp+shx+dbf+prj 多文件协调、GeoTIFF 的 TIFF 结构+GeoKeys+分块+金字塔）通过 GDAL FFI。

**API 设计**：

```moonbit
// ffi/gdal/gdal.mbt（概念设计）

/// 打开数据源（任意 GDAL 支持的格式）
pub fn open_dataset(path : String, format : String) -> GdalDataset raise GeoError {
  let h = GDALOpenEx(path, GDAL_OF_VECTOR, ..., ...)
  if h == RawPointer::null() {
    raise GeoError::FFIError("GDAL: failed to open \{path}")
  }
  { handle: h, }
}

/// 读取所有要素为 MoonBit Feature 数组
pub fn GdalDataset::read_features(self : GdalDataset) -> Array[@api.Feature] raise GeoError {
  // 遍历 Layer → Feature → Geometry + Attributes
  ...
}

/// 写入要素
pub fn GdalDataset::write_features(
  self : GdalDataset,
  features : Array[@api.Feature],
) -> Unit raise GeoError {
  ...
}
```

---

## 4. API 层 `api/`（公共门面）

### 4.1 职责

用户唯一接触的层。提供类型安全的 MoonBit API，内部根据操作类型自动路由到 `core/`（纯 MoonBit）或 `ffi/`（C 库绑定）。用户不需要知道某个操作是纯 MoonBit 还是 FFI 实现。

### 4.2 模块设计

```
api/
├── geometry.mbt    # 几何类型统一入口
├── ops.mbt         # 几何运算统一入口
├── crs.mbt         # CRS 变换统一入口
├── io.mbt          # 格式 I/O 统一入口
└── feature.mbt     # Feature = Geometry + Attributes（OGC 模型）
```

### 4.3 API 示例

```moonbit
// api/geometry.mbt — 几何类型
pub let p1 = Point2D::new(1.0, 2.0)
pub let poly = Polygon2D::from_vertices([(0.0, 0.0), (10.0, 0.0), (10.0, 10.0), (0.0, 10.0)])
pub let aabb = poly.bounding_box()  // → core

// api/ops.mbt — 几何运算（自动路由）
pub let inside = p1.in_polygon(poly)         // → core（纯 MoonBit）
pub let clipped = poly1.intersection(poly2)  // → ffi/geos（GEOS 布尔运算）
pub let buffered = poly.buffer(2.0)          // → ffi/geos（GEOS buffer）
pub let hull = points.convex_hull()          // → core（纯 MoonBit）

// api/crs.mbt — CRS 变换
pub let ctx = ProjContext::new()
pub let t = ProjTransform::from_epsg(ctx, 4326, 32650)  // WGS84 → UTM Zone 50N
pub let projected = t.transform([(116.0, 40.0)])        // → ffi/proj

// api/io.mbt — 格式 I/O（自动路由）
pub let geojson = GeoJSON::read_file("data.geojson")  // → core（纯 MoonBit）
pub let shp = Shapefile::read_file("data.shp")        // → ffi/gdal
pub let tiff = GeoTIFF::read_window("raster.tif", x, y, w, h)  // → ffi/gdal
```

### 4.4 路由策略

```moonbit
// api/ops.mbt 内部实现
pub fn Polygon2D::intersection(self, other : Polygon2D) -> Polygon2D raise GeoError {
  // 尝试 core 层快速路径（凸-凸裁剪）
  if self.is_convex() && other.is_convex() {
    return @core.clip.sutherland_hodgman(self, other)  // 纯 MoonBit，无 C 依赖
  }
  // 退化到 GEOS（任意多边形布尔运算）
  return @ffi.geos.intersection(self, other)  // FFI → GEOS
}
```

**关键设计**：core 层能做的优先用 core（无 C 依赖、WASM 友好），core 做不了的才 FFI。这保证了 core-only 构建也能覆盖常用场景。

---

## 5. WASM 构建策略

### 5.1 三种 WASM 包

| 包名 | 包含 | C 库编译方式 | 体积（估） | 典型用途 |
|------|------|------------|----------|---------|
| `moon-geo-core.wasm` | core/ + api/（geometry/ops 基础部分） | 无 C 依赖 | ~100KB | 嵌入式、教学、纯几何 |
| `moon-geo-proj.wasm` | core + ffi/proj | PROJ 编译为 WASM（Emscripten） | ~2MB | Web 地图、CRS 变换 |
| `moon-geo-full.wasm` | core + ffi/proj + ffi/geos + ffi/gdal | PROJ+GEOS+GDAL 编译为 WASM | ~10MB | 服务端 GIS、完整 I/O |

### 5.2 C 库编译到 WASM 的可行性

| C 库 | 已有 WASM 编译 | 注意事项 |
|------|--------------|---------|
| PROJ | ✅ [proj-wasm](https://github.com/mbtaffe/proj-wasm) 等社区项目 | 需打包 EPSG 数据库（~5MB） |
| GEOS | ✅ [geos-wasm](https://github.com/chrisdavies/geos-wasm) 等 | 体积适中，无文件系统依赖 |
| GDAL | ⚠️ 可编译但体积大 | 需选择性链接驱动（只编译需要的格式驱动） |

### 5.3 构建流程

```
build/
├── build-core.sh       # moon build --target wasm（无 C 依赖）
├── build-proj.sh       # 1. emcc PROJ → libproj.wasm
│                       # 2. moon build --target wasm + link libproj.wasm
├── build-full.sh       # 1. emcc PROJ+GEOS+GDAL → libs.wasm
│                       # 2. moon build --target wasm + link libs.wasm
└── CMakeLists.txt      # C 库的 Emscripten 构建配置
```

---

## 6. MoonBit FFI 调研清单（前置任务）

> **这是整个 FFI 架构的前置阻塞项。** 在写任何 FFI 绑定代码之前，必须先完成此调研。

### 6.1 需要验证的 MoonBit FFI 能力

| # | 调研项 | 验证方法 | 阻塞什么 |
|---|--------|---------|---------|
| 1 | MoonBit `extern "C"` 语法与调用约定 | 写一个 `extern "C" fn malloc(size: Int) -> RawPointer` 测试 | 全部 FFI |
| 2 | RawPointer 类型的可用操作（创建/解引用/null 判断） | 写最小 FFI demo | 全部 FFI |
| 3 | Drop trait / 资源释放机制（RAII） | 写一个持有 RawPointer 的 struct + Drop | RAII wrapper |
| 4 | C 结构体映射（如 PROJ 的 PJ_COORD） | 映射一个简单 C struct | PROJ/GEOS/GDAL |
| 5 | C 回调函数（GEOS 的 NoticeHandler 等） | 注册一个 MoonBit 函数为 C 回调 | GEOS 错误处理 |
| 6 | 批量数据传递（C 数组 ↔ MoonBit Array） | 传递 Double 数组给 C 函数 | PROJ 批量变换 |
| 7 | 错误跨边界传递（C errno / 返回码 → MoonBit raise） | C 函数返回错误码，MoonBit 转为 raise | 全部 FFI |
| 8 | moon.pkg 中声明 C 依赖 / 链接配置 | 配置一个依赖 libm 的包 | 构建系统 |
| 9 | WASM target 下 FFI 链接 C 库的方式 | 编译一个调用 libm 的 MoonBit 包到 WASM | WASM 构建 |
| 10 | Native target 下 FFI 链接系统库的方式 | 同上但 target=native | Native 构建 |

### 6.2 调研产出

每个调研项产出：
- 一个最小可运行的 MoonBit FFI demo（`ffi-spike/` 目录）
- 确认语法/机制/限制
- 记录到 `docs/ffi-research-report.md`

### 6.3 调研结论的分支

- **全部通过** → 按本文档推进 FFI 绑定
- **部分不通过** → 对不通过项设计 workaround（如手动 JSON 桥接、生成 C wrapper）
- **核心不通过**（无 FFI 或 FFI 不可用）→ 退回纯 MoonBit 方案，scope 进一步收缩到 core-only

---

## 7. 开发路线图（2–3 年到 1.0）

### 7.1 阶段总览

| 阶段 | 时间 | 版本 | 目标 | 交付物 |
|------|------|------|------|--------|
| Phase 0 | 月 1–2 | 0.2.0 | FFI 调研 + 代码重构 | FFI 调研报告 + core/ 目录重构 |
| Phase 1 | 月 3–6 | 0.3.0 | core 层完善 + API 门面 | core-only 可用、文档、测试 |
| Phase 2 | 月 7–12 | 0.4.0 | PROJ FFI 绑定 | CRS 全 EPSG、WASM with-proj |
| Phase 3 | 月 13–18 | 0.5.0 | GEOS FFI 绑定 | 布尔运算、buffer、DE-9IM |
| Phase 4 | 月 19–24 | 0.6.0 | GDAL FFI 绑定 | Shapefile/GeoTIFF/LAS I/O |
| Phase 5 | 月 25–30 | 1.0.0 | 稳定化 + 文档 + 性能基准 | 1.0 发布 |

### 7.2 Phase 0：FFI 调研 + 代码重构（月 1–2）

**任务**：
1. 完成 §6 FFI 调研清单全部 10 项
2. 将现有代码从 9 层洋葱结构迁移到 core/ 三层结构（§2.3 映射表）
3. 删除 atop/、space/（降级）、parallel/、domain/、eco/、draw/
4. 保留现有 668 个测试，确保迁移后全绿
5. 产出 `docs/ffi-research-report.md`

**验收**：`moon check` 0 errors、`moon test` 全绿、core/ 无 C 依赖

### 7.3 Phase 1：core 层完善 + API 门面（月 3–6）

**任务**：
1. 补全 core/geom 缺失类型：Polygon2D（含内孔）、MultiPolygon、Polyline
2. 补全 core/intersect：segment-segment、polygon-polygon 快速判定（SAT）
3. 补全 core/transform：Affine2D/3D 完整实现
4. 设计 api/ 门面层，统一入口
5. 补全白盒测试 + 黑盒测试
6. 写用户文档（快速上手 + API 参考）

**验收**：core-only WASM 可编译、api/ 层可用、测试 ≥ 800

### 7.4 Phase 2：PROJ FFI 绑定（月 7–12）

**任务**：
1. 实现 ffi/proj：ProjContext、ProjTransform、from_epsg、from_wkt、transform
2. RAII wrapper + 错误转换
3. api/crs 统一入口
4. 现有手写 UTM/Mercator 保留为 fallback（PROJ 不可用时降级）
5. PROJ 编译到 WASM（Emscripten）
6. 性能基准：PROJ FFI vs 手写 UTM，1M 点变换
7. 测试：与 PROJ C API 直接调用结果对比

**验收**：EPSG 全量可用、with-proj WASM 可编译、性能基准通过

### 7.5 Phase 3：GEOS FFI 绑定（月 13–18）

**任务**：
1. 实现 ffi/geos：GeosGeometry、from_polygon/to_polygon、intersection/union/difference/buffer
2. DE-9IM 空间关系：contains/within/intersects/touches/crosses/overlaps/disjoint
3. WKT/WKB 解析与序列化
4. api/ops 路由：core 快速路径 + GEOS 退化路径
5. GEOS 编译到 WASM
6. 测试：与 GEOS C API 结果对比 + 与 PostGIS 已知结果对比

**验收**：布尔运算可用、DE-9IM 可用、full WASM 可编译

### 7.6 Phase 4：GDAL FFI 绑定（月 19–24）

**任务**：
1. 实现 ffi/gdal：open_dataset、read_features、write_features
2. 支持格式：Shapefile、GeoTIFF（含分块/金字塔窗口读取）、LAS、KML、GML
3. api/io 统一入口：GeoJSON/STL → core，其余 → ffi/gdal
4. api/feature：OGC Feature 模型（Geometry + Attributes + CRS）
5. GDAL 选择性编译到 WASM（只链接需要的格式驱动）
6. 测试：与 GDAL C API 结果对比 + 与 ogrinfo/gdalinfo 输出对比

**验收**：Shapefile/GeoTIFF/LAS 读写可用

### 7.7 Phase 5：稳定化 + 1.0 发布（月 25–30）

**任务**：
1. API 审查 + 稳定化：冻结 1.0 API
2. 完整文档：快速上手、API 参考、迁移指南、性能基准报告
3. 三种 WASM 包发布到 npm（moon-geo-core / moon-geo-proj / moon-geo-full）
4. 性能基准：vs Turf.js / vs geos-wasm / vs Shapely
5. README 重写
6. 1.0.0 发布

**验收**：1.0 API 冻结、文档完整、三种 WASM 包可用、性能基准报告发布

---

## 8. 测试策略

### 8.1 三层测试

| 层 | 测试类型 | 工具 | 覆盖目标 |
|----|---------|------|---------|
| core/ | 白盒 + 黑盒 | `moon test`（`_wbtest.mbt` + `_test.mbt`） | 纯 MoonBit 逻辑正确性 |
| ffi/ | 对比测试 | `moon test` + C API 直接调用 | FFI 绑定与 C 库行为一致 |
| api/ | 集成测试 | `moon test` + 端到端 | 用户 API 可用性与文档一致性 |

### 8.2 FFI 对比测试策略

每个 FFI 绑定的测试模式：

```moonbit
test "PROJ UTM zone 50N matches known result" {
  let ctx = ProjContext::new()
  let t = ProjTransform::from_epsg(ctx, 4326, 32650)
  let result = t.transform([(116.391, 39.907)])  // 北京
  // 与 PROJ cs2cs 命令行工具输出对比
  assert_eq(result[0].0, 448357.69, tol=0.01)
  assert_eq(result[0].1, 4419873.22, tol=0.01)
}
```

### 8.3 回归测试数据

- WGS84 → UTM 已知坐标对（从 PROJ cs2cs 生成）
- 几何布尔运算已知结果（从 PostGIS 生成）
- Shapefile 读写往返（用 GDAL ogr2ogr 验证）
- GeoTIFF 窗口读取（用 GDAL gdalinfo 验证）

---

## 9. 错误处理

### 9.1 统一错误类型

```moonbit
pub enum GeoError {
  InvalidInput(String)       // 输入不合法（如负半径）
  OutOfBounds(String)        // 越界（如 UTM zone 61）
  Unsupported(String)        // 不支持的操作（如非凸多边形用 core 裁剪）
  FFIError(String)           // C 库返回错误
  FFINotAvailable(String)    // C 库未链接（core-only 构建调用需要 C 库的 API）
  ParseError(String)         // 格式解析失败
  IOError(String)            // 文件 I/O 失败
}
```

### 9.2 优雅降级

```moonbit
// api/ops.mbt
pub fn Polygon2D::intersection(self, other) -> Polygon2D raise GeoError {
  if self.is_convex() && other.is_convex() {
    return @core.clip.sutherland_hodgman(self, other)  // 纯 MoonBit
  }
  // 非凸多边形需要 GEOS
  // 如果 GEOS 未链接，返回明确错误而非 crash
  return @ffi.geos.intersection(self, other)  // 可能 raise FFINotAvailable
}
```

用户在 core-only 构建中调用需要 GEOS 的操作，得到 `FFINotAvailable` 错误，而非链接失败或 segfault。

---

## 10. 目录结构（最终）

```
moon-geo/
├── core/                       # 纯 MoonBit，无 C 依赖
│   ├── util/                   # num, cpx, vec, mat, quat
│   ├── geom/                   # point, line, plane, circle, polygon, aabb
│   ├── intersect/              # line-line, line-circle, segment-segment, point-polygon
│   ├── transform/              # affine, projective
│   ├── bvh/                    # BVH
│   ├── hemesh/                 # half-edge
│   ├── delaunay/               # Bowyer-Watson
│   ├── convex/                 # convex hull
│   └── clip/                   # Sutherland-Hodgman
│
├── ffi/                        # C 库 FFI 绑定
│   ├── proj/                   # PROJ: CRS 投影变换
│   ├── geos/                   # GEOS: 布尔运算, buffer, DE-9IM
│   └── gdal/                   # GDAL: 格式 I/O
│
├── api/                        # 公共 API 门面
│   ├── geometry.mbt            # 几何类型入口
│   ├── ops.mbt                 # 几何运算入口（路由 core/ffi）
│   ├── crs.mbt                 # CRS 入口（路由 ffi/proj）
│   ├── io.mbt                  # I/O 入口（路由 core/ffi/gdal）
│   └── feature.mbt             # OGC Feature 模型
│
├── build/                      # 构建脚本
│   ├── build-core.sh
│   ├── build-proj.sh
│   ├── build-full.sh
│   └── CMakeLists.txt
│
├── tests/                      # 集成测试 + 回归数据
│   ├── data/                   # 已知结果测试数据
│   └── regression/             # 回归测试
│
├── docs/
│   ├── 开发需求文档（FFI 绑定架构）.md  # 本文档
│   ├── ffi-research-report.md  # FFI 调研报告（Phase 0 产出）
│   ├── api-reference.md        # API 参考（Phase 5 产出）
│   └── benchmarks.md           # 性能基准报告
│
├── moon.mod                    # MoonBit 模块配置
└── README.md
```

---

## 11. 与 v2.0 立项文档的差异说明

### 11.1 架构差异

| 维度 | v2.0（原） | v3.0（FFI） |
|------|-----------|------------|
| 层数 | 9 层洋葱 + 5 层 platform = 14 层 | 3 层（core + ffi + api） |
| 实现方式 | 纯 MoonBit 自研全部 | 纯 MoonBit 基础 + FFI 绑定 C 库 |
| CRS | 手写 UTM/Mercator，计划全自研 | FFI → PROJ（全 EPSG、全投影） |
| 布尔运算 | 计划自研 | FFI → GEOS（工业验证） |
| 格式 I/O | 手写 JSON parser，计划自研全部 | 简单格式纯 MoonBit + 复杂格式 FFI → GDAL |
| 数学定位 | 复射影代数几何 + 代数拓扑 | 实用计算几何 |
| 抽象层 atop | 拓扑空间/流形/同调 | 删除（无实质实现） |
| 远期层 | parallel/domain/eco/draw | 删除（非目标） |

### 11.2 范围差异

| 维度 | v2.0 | v3.0 |
|------|------|------|
| 时间线 | 20 年（2026→2046） | 2.5 年（30 月到 1.0） |
| 版本目标 | 0.4.x.y → 1.0.0.0（20 年后） | 1.0.0（30 月后） |
| 对标 | CGAL + GDAL + ArcGIS | Shapely + WASM 友好 |
| 纪元 | 4 纪元 × 5 年 | 6 Phase × 5 月 |
| 治理 | BDFL → 委员会 → 基金会 → 标准组织 | 单人/小团队，1.0 后开放贡献 |
| 承诺 | 核心架构终身稳定 | 1.0 后 API 稳定 |

### 11.3 保留的部分

| 保留项 | 理由 |
|-------|------|
| 现有 util/ 全部代码 | 基础数值工具，纯 MoonBit，质量已验证 |
| 现有 alg/ 几何类型与求交 | 迁移到 core/geom + core/intersect |
| 现有 bv/bvh.mbt | 迁移到 core/bvh/ |
| 现有 brep/half_edge.mbt | 迁移到 core/hemesh/ |
| 现有 algo/triangulate.mbt | 迁移到 core/delaunay/ |
| 现有 algo/convex_hull.mbt | 迁移到 core/convex/ |
| 现有 platform/io/geojson | 迁移到 core/（纯 MoonBit，无 C 依赖） |
| 现有 platform/io/stl | 迁移到 core/（纯 MoonBit） |
| 现有 platform/crs/ 类型定义 | 保留为 api/ 用户类型（Ellipsoid/Datum/CrsKind） |
| AGENTS.md 工程规范 | 保留 CRAP/变异测试/白盒测试规则 |
| CI 配置 | 保留并扩展（增加 FFI 构建矩阵） |

### 11.4 删除的部分

| 删除项 | 理由 |
|-------|------|
| atop/（拓扑空间/流形/同调） | 无实质实现，不被依赖，范畴抽象非目标 |
| space/ 独立层 | 降级为 core/geom 中的类型，空间语义自带于几何类型 |
| algo/boolean.mbt 计划 | → ffi/geos |
| algo/buffer.mbt 计划 | → ffi/geos |
| algo/topo_relate.mbt 计划 | → ffi/geos |
| parallel/ | 远期非目标 |
| domain/（CAD/GIS/仿真/机器人适配） | 远期非目标 |
| eco/（语言绑定/生态接口） | WASM 即生态接口，不需要独立层 |
| draw/（可视化） | 非目标，可视化由用户端处理 |
| platform/pcloud/ | 远期非目标（需要时 FFI → PDAL） |
| platform/data/（Feature/Layer/Dataset） | 简化为 api/feature.mbt（OGC Feature 模型） |
| 20 年路线图 | 不可承诺 |
| 4 纪元治理演进 | 不现实 |
| "终身稳定"承诺 | 1.0 后稳定 |

---

## 12. 风险与缓解

| 风险 | 概率 | 影响 | 缓解 |
|------|------|------|------|
| MoonBit FFI 不可用或不成熟 | 中 | 高（阻塞全部 ffi/ 层） | Phase 0 先调研，不通过则退回 core-only |
| MoonBit 语言破坏性变更 | 中 | 中 | 0.x 阶段接受，1.0 后跟随 MoonBit 1.0 |
| PROJ/GEOS/GDAL WASM 编译困难 | 低 | 中 | 已有社区 WASM 编译先例；选择性链接控制体积 |
| C 库版本不兼容 | 低 | 低 | FFI 绑定声明最低版本，CI 矩阵覆盖 |
| 性能不如直接用 C 库 | 中 | 低 | MoonBit 层开销仅 API 调用 + 类型转换，基准验证 |
| WASM 体积过大 | 中 | 中 | 三种构建 profile，用户按需选择 |

---

## 13. 成功标准

### 13.1 1.0 发布标准

- [ ] core/ 层：2D/3D 几何类型 + 求交 + 变换 + BVH + 半边 + Delaunay + 凸包 + 裁剪，全部有测试
- [ ] ffi/proj：PROJ 绑定可用，EPSG 全量，WASM with-proj 可编译
- [ ] ffi/geos：GEOS 绑定可用，布尔运算 + buffer + DE-9IM，WASM full 可编译
- [ ] ffi/gdal：Shapefile + GeoTIFF + LAS 读写可用
- [ ] api/ 层：统一入口，类型安全，文档完整
- [ ] 三种 WASM 包可发布
- [ ] 性能基准报告：vs Turf.js / vs geos-wasm
- [ ] 测试 ≥ 1000，全绿
- [ ] API 参考 + 快速上手 + 迁移指南

### 13.2 不追求的标准

- ❌ CGAL 级精确谓词（远期，需 CGAL FFI）
- ❌ 全 OGC 标准符合（远期）
- ❌ 形式化验证（非目标）
- ❌ 20 年 API 稳定（1.0 后稳定，跟随语义版本）
- ❌ 全行业领域适配（非目标）

---

*本文档替代 v2.0《geo 统一空间数学引擎 · 开发立项技术规范》。v2.0 保留作为历史参考，但不再作为开发指导。*
