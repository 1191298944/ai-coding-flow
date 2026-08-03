# 若依框架代码模板

> 供 `ruoyi-dev` Skill 按需引用。生成表单/列表/后端分层代码时遵循本文件。

## 前端开发规范

### 1. Vue3 + Element Plus 表单模板
```vue
<template>
  <el-form :model="formData" :rules="rules" ref="formRef">
    <el-form-item label="字段名" prop="fieldName">
      <el-input v-model="formData.fieldName" />
    </el-form-item>
  </el-form>
</template>

<script setup lang="ts">
import { ref, reactive } from 'vue'
import type { FormInstance } from 'element-plus'

const formData = reactive({
  fieldName: ''
})

const rules = {
  fieldName: [{ required: true, message: '请输入字段名', trigger: 'blur' }]
}

const formRef = ref<FormInstance>()
</script>
```

### 2. vxe-table 集成模板
```vue
<template>
  <vxe-table
    :data="tableData"
    :loading="loading"
    :pagination="pagination"
    height="auto"
    virtual-scroll
    @page-change="handlePageChange"
  >
    <vxe-column field="fieldName" title="字段名" width="200" />
    <vxe-column field="createTime" title="创建时间" width="180" />
    <vxe-column title="操作" width="150">
      <template #default="{ row }">
        <el-button @click="handleEdit(row)">编辑</el-button>
      </template>
    </vxe-column>
  </vxe-table>
</template>

<script setup>
import { ref, reactive } from 'vue'

const tableData = ref([])
const loading = ref(false)
const pagination = reactive({
  currentPage: 1,
  pageSize: 20,
  total: 0
})

const handlePageChange = ({ currentPage, pageSize }) => {
  pagination.currentPage = currentPage
  pagination.pageSize = pageSize
  loadData()
}
</script>
```

### 3. 权限指令实现
```javascript
// Vue3权限指令
const hasPermi = {
  mounted(el, binding) {
    const { value } = binding
    const permissions = useUserStore().permissions
    
    if (value && !permissions.includes(value)) {
      el.parentNode?.removeChild(el)
    }
  }
}
```

### 4. 查询方法模板规范（前端）

- **适用范围**: 列表、树形、明细等「只读查询」的通用方法
- **统一响应结构**: 后端返回 `{ code, data, msg, total? }`，前端以 `code === 200` 作为成功判定
- **状态约定**:
  - 加载态变量命名使用 `xxxLoading`（如：`queryLoading`、`treeLoading`）
  - 数据变量命名使用 `list`/`data`/`treeData` 等符合语义的名称
  - 分页变量包含 `pageNum`、`pageSize`、`total`
- **异常与空值处理**:
  - 失败或异常时，数据回落到安全默认值（数组用 `[]`，对象用 `{}`）
  - 统一通过 `proxy?.$modal?.msgError(res?.msg || '查询失败')` 提示
- **并发与幂等**:
  - 重入保护：查询进行中直接 `return`
  - 可选取消：使用 `AbortController` 取消上一次未完成的请求（Axios ≥ 1.x 支持）
- **可扩展性**: 模板支持分页/条件查询/去抖节流等增强能力

```ts
// 查询方法标准模板（Vue3 + <script setup> + Axios）
// 代码注释与文档统一使用中文
import { ref, getCurrentInstance, onBeforeUnmount } from 'vue'
// import { listXxx } from '@/api/xxx' // 按模块引入实际API

const { proxy } = getCurrentInstance() as any

// 加载与数据状态
const queryLoading = ref(false)
const list = ref<any[]>([])
const total = ref(0)

// 可选：用于取消上一次请求（Axios >= 1.x 支持 AbortController）
let abortController: AbortController | null = null

// 查询条件——按具体业务补充字段
const queryForm = ref<Record<string, any>>({})

const handleQuery = () => {
  getList()
}

const getList = async () => {
  // 重入保护
  if (queryLoading.value) return
  queryLoading.value = true

  // 取消上一次未完成的请求（可选）
  if (abortController) {
    try { abortController.abort() } catch {}
  }
  abortController = new AbortController()

  try {
    // 将 listXxx 替换为具体 API 方法；第二参透传 signal 以支持取消
    const res = await listXxx(queryForm.value, { signal: abortController.signal })

    if (res && res.code === 200) {
      // 兼容多种数据结构：data.rows/data/list/纯数组
      const rows = Array.isArray(res.rows) ? res.rows : Array.isArray(res.data) ? res.data : []
      const totalCount = res.total && !isNaN(res.total) ? res.total : rows.length

      list.value = rows
      total.value = totalCount
    } else {
      list.value = []
      total.value = 0
      proxy?.$modal?.msgError(res?.msg || '查询失败')
    }
  } catch (err) {
    list.value = []
    total.value = 0
    proxy?.$modal?.msgError('查询失败')
  } finally {
    queryLoading.value = false
  }
}

onBeforeUnmount(() => {
  if (abortController) {
    try { abortController.abort() } catch {}
    abortController = null
  }
})
```

```ts
// 树数据查询示例
import { ref, getCurrentInstance } from 'vue'
import { getXxxTree } from '@/api/xxx/xxx' // 替换为实际 API 路径

const { proxy } = getCurrentInstance() as any
const treeLoading = ref(false)
const treeData = ref<any[]>([])

const queryTree = async () => {
  if (treeLoading.value) return
  treeLoading.value = true
  try {
    const res = await getXxxTree()
    if (res && res.code === 200) {
      treeData.value = Array.isArray(res.data) ? res.data : []
    } else {
      treeData.value = []
      proxy?.$modal?.msgError(res?.msg || '查询失败')
    }
  } catch (e) {
    treeData.value = []
    proxy?.$modal?.msgError('查询失败')
  } finally {
    treeLoading.value = false
  }
}
```

「落地检查清单」
- 成功判定是否只以 `code === 200` 为准
- 失败/异常是否回落到安全默认值并提示错误
- 是否存在并发重入保护；如有需要是否支持取消上一次请求
- 是否按页面语义命名 `xxxLoading`、`list/treeData`、`total`
- 是否为分页与条件查询预留 `queryForm` 扩展位

## 后端开发规范

### 1. Entity 实体类模板
```java
import com.baomidou.mybatisplus.annotation.TableId;
import com.baomidou.mybatisplus.annotation.TableName;
import com.yuwen.common.core.domain.BaseEntity;
import lombok.Data;
import lombok.EqualsAndHashCode;

/**
 * xxx实体类
 */
@Data
@EqualsAndHashCode(callSuper = true)
@TableName("xxx_table")
public class XxxEntity extends BaseEntity {

    @TableId
    private Long id;

    /** 字段说明 */
    private String fieldName;
}
```

### 2. Mapper 接口模板
```java
import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import org.apache.ibatis.annotations.Mapper;

/**
 * xxx Mapper接口
 */
@Mapper
public interface XxxMapper extends BaseMapper<XxxEntity> {

    /**
     * 自定义查询（复杂SQL在XML中实现）
     */
    List<XxxEntity> selectXxxList(XxxQueryParam param);
}
```

### 3. Service 接口模板
```java
import com.baomidou.mybatisplus.extension.service.IService;

/**
 * xxx Service接口
 */
public interface IXxxService extends IService<XxxEntity> {

    /**
     * 查询列表
     */
    List<XxxEntity> queryList(XxxQueryParam param);

    /**
     * 新增
     */
    void createXxx(XxxEntity entity);

    /**
     * 修改
     */
    void updateXxx(XxxEntity entity);

    /**
     * 删除
     */
    void deleteXxx(Long[] ids);
}
```

### 4. Service 实现类模板（含接口拆分示范）
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import com.yuwen.common.exception.ServiceException;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;

/**
 * xxx Service实现
 */
@Service
public class XxxServiceImpl extends ServiceImpl<XxxMapper, XxxEntity> implements IXxxService {

    private static final Logger log = LoggerFactory.getLogger(XxxServiceImpl.class);

    @Override
    public void createXxx(XxxEntity entity) {
        validateXxx(entity);
        enrichXxx(entity);
        save(entity);
    }

    @Override
    public void updateXxx(XxxEntity entity) {
        validateXxx(entity);
        updateById(entity);
    }

    @Override
    public void deleteXxx(Long[] ids) {
        removeByIds(Arrays.asList(ids));
    }

    /** 校验实体合法性（卫语句，不嵌套） */
    private void validateXxx(XxxEntity entity) {
        if (entity == null) throw new ServiceException("参数不能为空");
        if (StringUtils.isBlank(entity.getFieldName())) throw new ServiceException("字段名不能为空");
    }

    /** 补充默认值或计算字段（与校验分离） */
    private void enrichXxx(XxxEntity entity) {
        // 补充默认值逻辑
    }
}
```

### 5. Controller 模板
```java
import org.springframework.web.bind.annotation.*;
import com.yuwen.common.core.controller.BaseController;
import com.yuwen.common.core.domain.R;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;

/**
 * xxx Controller
 */
@Tag(name = "xxx管理")
@RestController
@RequestMapping("/xxx")
public class XxxController extends BaseController {

    @Autowired
    private IXxxService xxxService;

    @Operation(summary = "查询列表")
    @GetMapping("/list")
    public R<List<XxxEntity>> list(XxxQueryParam param) {
        return R.ok(xxxService.queryList(param));
    }

    @Operation(summary = "新增")
    @PostMapping
    public R<Void> add(@RequestBody @Validated XxxEntity entity) {
        xxxService.createXxx(entity);
        return R.ok();
    }

    @Operation(summary = "修改")
    @PutMapping
    public R<Void> edit(@RequestBody @Validated XxxEntity entity) {
        xxxService.updateXxx(entity);
        return R.ok();
    }

    @Operation(summary = "删除")
    @DeleteMapping("/{ids}")
    public R<Void> remove(@PathVariable Long[] ids) {
        xxxService.deleteXxx(ids);
        return R.ok();
    }
}
```
