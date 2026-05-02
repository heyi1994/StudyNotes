# Ant Design

Ant Design（antd）是蚂蚁集团出品的企业级 React UI 组件库，内置 60+ 高质量组件，开箱即用，特别适合中后台管理系统。当前主流版本为 **v5**，基于 CSS-in-JS 实现主题系统。

---

## 安装

```shell
npm install antd
```

v5 不需要额外引入 CSS 文件，开箱即用：

```tsx
// main.tsx 或 App.tsx
import { Button } from "antd"

export default function App() {
  return <Button type="primary">Hello Antd</Button>
}
```

### 按需加载

v5 默认支持 Tree Shaking，直接按需 import 即可，无需额外配置：

```tsx
import { Button, Input, Table } from "antd"
```

---

## 主题定制

### ConfigProvider 全局配置

`ConfigProvider` 是 antd 的全局配置入口，包裹在应用根部：

```tsx
// App.tsx
import { ConfigProvider } from "antd"
import zhCN from "antd/locale/zh_CN"

export default function App() {
  return (
    <ConfigProvider
      locale={zhCN}
      theme={{
        token: {
          // 主色
          colorPrimary: "#1677ff",
          colorSuccess: "#52c41a",
          colorWarning: "#faad14",
          colorError: "#ff4d4f",
          colorInfo: "#1677ff",

          // 圆角
          borderRadius: 6,

          // 字体
          fontSize: 14,
          fontFamily: "'PingFang SC', 'Microsoft YaHei', sans-serif",

          // 线条
          lineWidth: 1,

          // 间距
          padding: 16,
          paddingLG: 24,
          paddingSM: 12,
        },
        components: {
          // 覆盖单个组件的 token
          Button: {
            colorPrimary: "#00b96b",
            borderRadius: 8,
            controlHeight: 40,
          },
          Table: {
            borderRadius: 8,
            headerBg: "#f5f5f5",
          },
        },
      }}
    >
      <RouterProvider router={router} />
    </ConfigProvider>
  )
}
```

### 暗色模式

```tsx
import { ConfigProvider, theme } from "antd"

const { darkAlgorithm, compactAlgorithm, defaultAlgorithm } = theme

function App() {
  const [isDark, setIsDark] = useState(false)

  return (
    <ConfigProvider
      theme={{
        algorithm: isDark ? darkAlgorithm : defaultAlgorithm,
        // 同时开启暗色 + 紧凑
        // algorithm: [darkAlgorithm, compactAlgorithm],
      }}
    >
      <button onClick={() => setIsDark(!isDark)}>切换主题</button>
      <YourApp />
    </ConfigProvider>
  )
}
```

### useToken：在组件中读取主题变量

```tsx
import { theme } from "antd"

function MyComponent() {
  const { token } = theme.useToken()

  return (
    <div
      style={{
        background: token.colorBgContainer,
        borderRadius: token.borderRadius,
        padding: token.padding,
        color: token.colorText,
      }}
    >
      使用主题变量
    </div>
  )
}
```

---

## 布局组件

### Layout

```tsx
import { Layout, Menu, theme } from "antd"

const { Header, Content, Footer, Sider } = Layout

function AdminLayout() {
  const [collapsed, setCollapsed] = useState(false)
  const { token } = theme.useToken()

  return (
    <Layout style={{ minHeight: "100vh" }}>
      {/* 侧边栏 */}
      <Sider
        collapsible
        collapsed={collapsed}
        onCollapse={setCollapsed}
        theme="light"
        width={220}
      >
        <div style={{ height: 32, margin: 16, background: "rgba(0,0,0,.1)", borderRadius: 6 }} />
        <Menu
          mode="inline"
          defaultSelectedKeys={["1"]}
          items={[
            { key: "1", icon: <DashboardOutlined />, label: "仪表盘" },
            { key: "2", icon: <UserOutlined />, label: "用户管理" },
            {
              key: "sub1",
              icon: <SettingOutlined />,
              label: "系统设置",
              children: [
                { key: "3", label: "基本设置" },
                { key: "4", label: "权限管理" },
              ],
            },
          ]}
        />
      </Sider>

      <Layout>
        <Header style={{ padding: "0 24px", background: token.colorBgContainer, display: "flex", alignItems: "center" }}>
          <h2 style={{ margin: 0 }}>管理系统</h2>
        </Header>
        <Content style={{ margin: "24px 16px", padding: 24, background: token.colorBgContainer, borderRadius: token.borderRadius }}>
          <Outlet />
        </Content>
        <Footer style={{ textAlign: "center" }}>©2025 My Company</Footer>
      </Layout>
    </Layout>
  )
}
```

### Grid 栅格

antd 栅格系统基于 24 列：

```tsx
import { Row, Col } from "antd"

{/* 基础用法：24 列 */}
<Row gutter={16}>
  <Col span={8}>占 8 列（1/3）</Col>
  <Col span={8}>占 8 列（1/3）</Col>
  <Col span={8}>占 8 列（1/3）</Col>
</Row>

{/* 响应式 */}
<Row gutter={{ xs: 8, sm: 16, md: 24 }}>
  <Col xs={24} sm={12} md={8} lg={6}>响应式列</Col>
  <Col xs={24} sm={12} md={8} lg={6}>响应式列</Col>
  <Col xs={24} sm={12} md={8} lg={6}>响应式列</Col>
  <Col xs={24} sm={12} md={8} lg={6}>响应式列</Col>
</Row>

{/* 偏移 */}
<Row>
  <Col span={8}>左侧</Col>
  <Col span={8} offset={8}>右侧（偏移 8 列）</Col>
</Row>

{/* Flex 对齐 */}
<Row justify="space-between" align="middle">
  <Col span={4}>左</Col>
  <Col span={4}>中</Col>
  <Col span={4}>右</Col>
</Row>
```

### Space 间距

```tsx
import { Space, Divider } from "antd"

{/* 水平间距 */}
<Space size="middle">
  <Button>按钮一</Button>
  <Button>按钮二</Button>
  <Button>按钮三</Button>
</Space>

{/* 垂直间距 */}
<Space direction="vertical" size={16} style={{ display: "flex" }}>
  <Input placeholder="用户名" />
  <Input placeholder="密码" />
  <Button type="primary" block>登录</Button>
</Space>

{/* 自动换行 */}
<Space size={[8, 16]} wrap>
  {tags.map(tag => <Tag key={tag}>{tag}</Tag>)}
</Space>

{/* 带分隔符 */}
<Space split={<Divider type="vertical" />}>
  <a>编辑</a>
  <a>删除</a>
  <a>查看</a>
</Space>
```

---

## 通用组件

### Button 按钮

```tsx
import { Button, Space } from "antd"

{/* 类型 */}
<Button type="primary">主要按钮</Button>
<Button type="default">默认按钮</Button>
<Button type="dashed">虚线按钮</Button>
<Button type="text">文字按钮</Button>
<Button type="link">链接按钮</Button>

{/* 危险状态 */}
<Button danger>危险</Button>
<Button type="primary" danger>主要危险</Button>

{/* 尺寸 */}
<Button size="large">大</Button>
<Button size="middle">中（默认）</Button>
<Button size="small">小</Button>

{/* 状态 */}
<Button loading>加载中</Button>
<Button disabled>禁用</Button>
<Button block>通栏按钮</Button>

{/* 图标 */}
<Button icon={<SearchOutlined />}>搜索</Button>
<Button type="primary" shape="circle" icon={<PlusOutlined />} />
<Button type="primary" shape="round" icon={<DownloadOutlined />}>下载</Button>
```

### Typography 排版

```tsx
import { Typography } from "antd"
const { Title, Text, Paragraph, Link } = Typography

<Typography>
  <Title>一级标题</Title>
  <Title level={2}>二级标题</Title>
  <Title level={3}>三级标题</Title>

  <Paragraph>
    这是一段普通段落文字，<Text strong>加粗</Text>、
    <Text italic>斜体</Text>、<Text underline>下划线</Text>、
    <Text delete>删除线</Text>、<Text code>行内代码</Text>、
    <Text type="secondary">次要文字</Text>、
    <Text type="success">成功</Text>、
    <Text type="warning">警告</Text>、
    <Text type="danger">危险</Text>。
  </Paragraph>

  {/* 可复制 */}
  <Paragraph copyable>可复制的文本内容</Paragraph>
  <Paragraph copyable={{ text: "自定义复制内容" }}>自定义复制</Paragraph>

  {/* 省略 */}
  <Paragraph ellipsis={{ rows: 2, expandable: true, symbol: "展开" }}>
    这是一段很长的文字，超过两行后会自动折叠，点击展开可以查看全部内容...
  </Paragraph>

  {/* 可编辑 */}
  <Paragraph editable={{ onChange: (val) => console.log(val) }}>
    可以双击编辑这段文字
  </Paragraph>

  <Link href="https://ant.design" target="_blank">Ant Design 官网</Link>
</Typography>
```

### Icon 图标

antd v5 使用 `@ant-design/icons`：

```shell
npm install @ant-design/icons
```

```tsx
import {
  UserOutlined, SearchOutlined, PlusOutlined,
  EditOutlined, DeleteOutlined, SettingOutlined,
  CheckCircleOutlined, CloseCircleOutlined,
  LoadingOutlined, DownloadOutlined, UploadOutlined,
} from "@ant-design/icons"

<UserOutlined />
<SearchOutlined style={{ fontSize: 20, color: "#1677ff" }} />

{/* 旋转动画 */}
<LoadingOutlined spin />

{/* 自定义图标 */}
import Icon from "@ant-design/icons"
const HeartSvg = () => <svg>...</svg>
<Icon component={HeartSvg} />
```

---

## 数据录入

### Form 表单

antd Form 是最常用也最复杂的组件之一：

```tsx
import { Form, Input, Button, Select, Checkbox, DatePicker, InputNumber, Radio, Switch, Upload } from "antd"

interface FormValues {
  username: string
  password: string
  email: string
  role: string
  age: number
  agree: boolean
}

function RegisterForm() {
  const [form] = Form.useForm<FormValues>()

  async function onFinish(values: FormValues) {
    console.log("提交：", values)
    await submitForm(values)
  }

  return (
    <Form
      form={form}
      layout="vertical"           // horizontal | vertical | inline
      onFinish={onFinish}
      onFinishFailed={({ errorFields }) => console.log("验证失败：", errorFields)}
      initialValues={{ role: "user", agree: false }}
      autoComplete="off"
    >
      <Form.Item
        name="username"
        label="用户名"
        rules={[
          { required: true, message: "请输入用户名" },
          { min: 2, message: "至少 2 个字符" },
          { max: 20, message: "最多 20 个字符" },
          { pattern: /^[a-zA-Z0-9_]+$/, message: "只能包含字母、数字、下划线" },
        ]}
      >
        <Input placeholder="请输入用户名" prefix={<UserOutlined />} />
      </Form.Item>

      <Form.Item
        name="password"
        label="密码"
        rules={[
          { required: true, message: "请输入密码" },
          { min: 8, message: "密码至少 8 位" },
        ]}
      >
        <Input.Password placeholder="请输入密码" />
      </Form.Item>

      <Form.Item
        name="confirm"
        label="确认密码"
        dependencies={["password"]}
        rules={[
          { required: true, message: "请确认密码" },
          ({ getFieldValue }) => ({
            validator(_, value) {
              if (!value || getFieldValue("password") === value) {
                return Promise.resolve()
              }
              return Promise.reject(new Error("两次密码不一致"))
            },
          }),
        ]}
      >
        <Input.Password placeholder="请再次输入密码" />
      </Form.Item>

      <Form.Item name="email" label="邮箱" rules={[{ type: "email", message: "邮箱格式不正确" }, { required: true }]}>
        <Input placeholder="name@example.com" />
      </Form.Item>

      <Form.Item name="role" label="角色">
        <Select
          options={[
            { value: "admin", label: "管理员" },
            { value: "user", label: "普通用户" },
            { value: "guest", label: "访客" },
          ]}
        />
      </Form.Item>

      <Form.Item name="age" label="年龄">
        <InputNumber min={1} max={120} style={{ width: "100%" }} />
      </Form.Item>

      <Form.Item name="agree" valuePropName="checked" rules={[{ validator: (_, v) => v ? Promise.resolve() : Promise.reject("请阅读并同意协议") }]}>
        <Checkbox>我已阅读并同意<a href="#">用户协议</a></Checkbox>
      </Form.Item>

      <Form.Item>
        <Button type="primary" htmlType="submit" block>注册</Button>
      </Form.Item>
    </Form>
  )
}
```

### Form 动态字段

```tsx
<Form.List name="phones">
  {(fields, { add, remove }) => (
    <>
      {fields.map(({ key, name, ...restField }) => (
        <Space key={key} align="baseline">
          <Form.Item
            {...restField}
            name={[name, "type"]}
            rules={[{ required: true, message: "请选择类型" }]}
          >
            <Select options={[{ value: "mobile", label: "手机" }, { value: "work", label: "工作" }]} style={{ width: 100 }} />
          </Form.Item>
          <Form.Item
            {...restField}
            name={[name, "number"]}
            rules={[{ required: true, message: "请输入电话" }]}
          >
            <Input placeholder="电话号码" />
          </Form.Item>
          <MinusCircleOutlined onClick={() => remove(name)} />
        </Space>
      ))}
      <Form.Item>
        <Button type="dashed" onClick={() => add()} icon={<PlusOutlined />} block>
          添加电话
        </Button>
      </Form.Item>
    </>
  )}
</Form.List>
```

### Form 命令式操作

```tsx
const [form] = Form.useForm()

// 设置字段值
form.setFieldValue("username", "张三")
form.setFieldsValue({ username: "张三", email: "zhang@example.com" })

// 读取字段值
const username = form.getFieldValue("username")
const all = form.getFieldsValue()

// 手动触发校验
const values = await form.validateFields()
await form.validateFields(["username", "email"])  // 只校验指定字段

// 重置表单
form.resetFields()
form.resetFields(["username"])

// 设置字段错误
form.setFields([{ name: "username", errors: ["用户名已存在"] }])
```

### Input 输入框

```tsx
import { Input } from "antd"

<Input placeholder="普通输入框" />
<Input.Password placeholder="密码输入框" />
<Input.TextArea rows={4} placeholder="多行文本" maxLength={200} showCount />
<Input.Search placeholder="搜索" onSearch={(val) => console.log(val)} enterButton />

{/* 前缀/后缀 */}
<Input prefix={<UserOutlined />} suffix={<Tooltip title="提示"><InfoCircleOutlined /></Tooltip>} />

{/* 前置/后置标签 */}
<Input addonBefore="http://" addonAfter=".com" placeholder="网址" />
<Input addonBefore={
  <Select defaultValue="https://" style={{ width: 90 }} options={[{ value: "http://" }, { value: "https://" }]} />
} />
```

### Select 选择器

```tsx
import { Select } from "antd"

{/* 基础 */}
<Select
  style={{ width: 200 }}
  placeholder="请选择"
  options={[
    { value: "1", label: "选项一" },
    { value: "2", label: "选项二", disabled: true },
  ]}
  onChange={(value, option) => console.log(value, option)}
/>

{/* 多选 */}
<Select mode="multiple" allowClear style={{ width: "100%" }} placeholder="可多选" options={options} />

{/* 标签模式（可自由输入） */}
<Select mode="tags" style={{ width: "100%" }} placeholder="输入后回车添加标签" />

{/* 可搜索 */}
<Select showSearch filterOption={(input, option) => (option?.label ?? "").toLowerCase().includes(input.toLowerCase())} options={options} />

{/* 远程搜索 */}
<Select
  showSearch
  filterOption={false}
  onSearch={fetchOptions}    // 输入时请求
  notFoundContent={loading ? <Spin size="small" /> : null}
  options={options}
/>

{/* 分组 */}
<Select options={[
  { label: "一组", options: [{ value: "a", label: "A" }, { value: "b", label: "B" }] },
  { label: "二组", options: [{ value: "c", label: "C" }] },
]} />
```

### DatePicker 日期选择

```tsx
import { DatePicker, Space } from "antd"
import dayjs from "dayjs"

<DatePicker onChange={(date, dateStr) => console.log(dateStr)} />
<DatePicker.RangePicker />
<DatePicker showTime placeholder="选择日期时间" />

{/* 格式化 */}
<DatePicker format="YYYY年MM月DD日" />

{/* 预设范围 */}
<DatePicker.RangePicker
  presets={[
    { label: "最近 7 天", value: [dayjs().subtract(7, "d"), dayjs()] },
    { label: "最近 30 天", value: [dayjs().subtract(30, "d"), dayjs()] },
    { label: "本月", value: [dayjs().startOf("month"), dayjs().endOf("month")] },
  ]}
/>

{/* 禁用日期 */}
<DatePicker disabledDate={(current) => current && current < dayjs().startOf("day")} />
```

### Upload 上传

```tsx
import { Upload, Button, message } from "antd"
import type { UploadProps } from "antd"

const props: UploadProps = {
  name: "file",
  action: "/api/upload",
  headers: { authorization: `Bearer ${token}` },
  accept: "image/*",
  maxCount: 3,
  onChange({ file, fileList }) {
    if (file.status === "done") {
      message.success(`${file.name} 上传成功`)
    } else if (file.status === "error") {
      message.error(`${file.name} 上传失败`)
    }
  },
  beforeUpload(file) {
    const isLt2M = file.size / 1024 / 1024 < 2
    if (!isLt2M) message.error("图片不能超过 2MB")
    return isLt2M
  },
}

{/* 点击上传 */}
<Upload {...props}>
  <Button icon={<UploadOutlined />}>选择文件</Button>
</Upload>

{/* 拖拽上传 */}
<Upload.Dragger {...props}>
  <p className="ant-upload-drag-icon"><InboxOutlined /></p>
  <p>点击或拖拽文件到此区域上传</p>
  <p className="ant-upload-hint">支持单个或批量上传</p>
</Upload.Dragger>

{/* 图片上传预览 */}
<Upload
  listType="picture-card"
  fileList={fileList}
  onPreview={handlePreview}
  onChange={({ fileList }) => setFileList(fileList)}
>
  {fileList.length < 8 && <div><PlusOutlined /><div>上传</div></div>}
</Upload>
```

---

## 数据展示

### Table 表格

```tsx
import { Table, Tag, Space, Button, Popconfirm } from "antd"
import type { TableProps, TableColumnsType } from "antd"

interface User {
  key: string
  name: string
  age: number
  role: string
  status: "active" | "inactive"
  createdAt: string
}

const columns: TableColumnsType<User> = [
  {
    title: "姓名",
    dataIndex: "name",
    key: "name",
    sorter: (a, b) => a.name.localeCompare(b.name),
    render: (name) => <a>{name}</a>,
  },
  {
    title: "年龄",
    dataIndex: "age",
    key: "age",
    sorter: (a, b) => a.age - b.age,
    width: 80,
  },
  {
    title: "角色",
    dataIndex: "role",
    key: "role",
    filters: [
      { text: "管理员", value: "admin" },
      { text: "普通用户", value: "user" },
    ],
    onFilter: (value, record) => record.role === value,
  },
  {
    title: "状态",
    dataIndex: "status",
    key: "status",
    render: (status) => (
      <Tag color={status === "active" ? "green" : "red"}>
        {status === "active" ? "正常" : "禁用"}
      </Tag>
    ),
  },
  {
    title: "操作",
    key: "action",
    fixed: "right",
    width: 160,
    render: (_, record) => (
      <Space>
        <Button type="link" size="small" onClick={() => handleEdit(record)}>编辑</Button>
        <Popconfirm
          title="确认删除？"
          description="删除后无法恢复"
          onConfirm={() => handleDelete(record.key)}
          okText="确认"
          cancelText="取消"
        >
          <Button type="link" size="small" danger>删除</Button>
        </Popconfirm>
      </Space>
    ),
  },
]

function UserTable() {
  const [selectedRowKeys, setSelectedRowKeys] = useState<React.Key[]>([])
  const [pagination, setPagination] = useState({ current: 1, pageSize: 10, total: 0 })

  return (
    <Table
      columns={columns}
      dataSource={users}
      rowKey="key"
      loading={loading}
      scroll={{ x: 1000 }}
      pagination={{
        ...pagination,
        showSizeChanger: true,
        showQuickJumper: true,
        showTotal: (total) => `共 ${total} 条`,
        onChange: (page, pageSize) => fetchUsers({ page, pageSize }),
      }}
      rowSelection={{
        selectedRowKeys,
        onChange: setSelectedRowKeys,
        selections: [Table.SELECTION_ALL, Table.SELECTION_NONE],
      }}
      summary={(data) => (
        <Table.Summary.Row>
          <Table.Summary.Cell index={0} colSpan={2}>汇总</Table.Summary.Cell>
          <Table.Summary.Cell index={2}>
            <Text type="danger">{data.reduce((sum, r) => sum + r.age, 0)}</Text>
          </Table.Summary.Cell>
        </Table.Summary.Row>
      )}
    />
  )
}
```

### List 列表

```tsx
import { List, Avatar, Card } from "antd"

{/* 基础列表 */}
<List
  itemLayout="horizontal"
  dataSource={users}
  loading={loading}
  pagination={{ pageSize: 10 }}
  renderItem={(item) => (
    <List.Item
      actions={[<a key="edit">编辑</a>, <a key="delete">删除</a>]}
    >
      <List.Item.Meta
        avatar={<Avatar src={item.avatar} />}
        title={<a href="#">{item.name}</a>}
        description={item.email}
      />
      <div>{item.role}</div>
    </List.Item>
  )}
/>

{/* 卡片列表（Grid 模式） */}
<List
  grid={{ gutter: 16, xs: 1, sm: 2, md: 3, lg: 4 }}
  dataSource={items}
  renderItem={(item) => (
    <List.Item>
      <Card title={item.title}>{item.description}</Card>
    </List.Item>
  )}
/>
```

### Descriptions 描述列表

```tsx
import { Descriptions, Badge } from "antd"
import type { DescriptionsProps } from "antd"

const items: DescriptionsProps["items"] = [
  { key: "1", label: "用户名",   children: "张三" },
  { key: "2", label: "邮箱",     children: "zhang@example.com" },
  { key: "3", label: "角色",     children: "管理员" },
  { key: "4", label: "手机",     children: "138****8888" },
  { key: "5", label: "状态",     children: <Badge status="processing" text="正常" /> },
  { key: "6", label: "注册时间", children: "2024-01-01 10:00:00" },
  { key: "7", label: "备注",     children: "这是一段较长的备注内容", span: 3 },
]

<Descriptions
  title="用户详情"
  bordered
  column={{ xs: 1, sm: 2, md: 3 }}
  items={items}
  extra={<Button type="primary">编辑</Button>}
/>
```

### Statistic 统计数值

```tsx
import { Statistic, Card, Row, Col } from "antd"
import { ArrowUpOutlined, ArrowDownOutlined } from "@ant-design/icons"
import CountUp from "react-countup"

<Row gutter={16}>
  <Col span={6}>
    <Card>
      <Statistic title="活跃用户" value={11280} prefix={<UserOutlined />} />
    </Card>
  </Col>
  <Col span={6}>
    <Card>
      <Statistic title="今日收入" value={9280} precision={2} prefix="¥" suffix="元"
        valueStyle={{ color: "#3f8600" }}
        prefix={<ArrowUpOutlined />}
      />
    </Card>
  </Col>
  <Col span={6}>
    <Card>
      <Statistic title="退款率" value={2.3} precision={1} suffix="%" valueStyle={{ color: "#cf1322" }} prefix={<ArrowDownOutlined />} />
    </Card>
  </Col>
  <Col span={6}>
    <Card>
      {/* 动画数字 */}
      <Statistic title="总订单" value={112893} formatter={(v) => <CountUp end={Number(v)} duration={2} separator="," />} />
    </Card>
  </Col>
</Row>
```

### Tag 标签

```tsx
import { Tag, Space } from "antd"

{/* 基础 */}
<Tag>默认</Tag>
<Tag color="magenta">品红</Tag>
<Tag color="red">红色</Tag>
<Tag color="orange">橙色</Tag>
<Tag color="green">绿色</Tag>
<Tag color="blue">蓝色</Tag>
<Tag color="purple">紫色</Tag>

{/* 状态色 */}
<Tag color="success">成功</Tag>
<Tag color="processing">处理中</Tag>
<Tag color="error">错误</Tag>
<Tag color="warning">警告</Tag>

{/* 可关闭 */}
<Tag closable onClose={() => handleClose()}>可关闭标签</Tag>

{/* 动态增删 */}
{tags.map(tag => <Tag key={tag} closable onClose={() => removeTag(tag)}>{tag}</Tag>)}
<Input size="small" style={{ width: 80 }} onPressEnter={(e) => addTag(e.currentTarget.value)} />
```

### Progress 进度条

```tsx
import { Progress } from "antd"

<Progress percent={60} />
<Progress percent={100} status="success" />
<Progress percent={70} status="exception" />
<Progress percent={50} showInfo={false} />

{/* 环形 */}
<Progress type="circle" percent={75} />
<Progress type="circle" percent={100} status="success" />
<Progress type="circle" percent={70} format={(percent) => `${percent}分`} />

{/* 仪表盘 */}
<Progress type="dashboard" percent={75} />

{/* 步骤进度 */}
<Progress percent={60} steps={5} />

{/* 渐变色 */}
<Progress percent={80} strokeColor={{ "0%": "#108ee9", "100%": "#87d068" }} />
```

### Tree 树形控件

```tsx
import { Tree } from "antd"
import type { TreeDataNode } from "antd"

const treeData: TreeDataNode[] = [
  {
    title: "技术部",
    key: "0",
    children: [
      {
        title: "前端组",
        key: "0-0",
        children: [
          { title: "张三", key: "0-0-0", isLeaf: true },
          { title: "李四", key: "0-0-1", isLeaf: true },
        ],
      },
      { title: "后端组", key: "0-1", children: [{ title: "王五", key: "0-1-0", isLeaf: true }] },
    ],
  },
]

<Tree
  checkable
  defaultExpandAll
  treeData={treeData}
  onCheck={(checked) => console.log(checked)}
  onSelect={(selected, { node }) => console.log(node.title)}
/>
```

---

## 反馈组件

### Modal 对话框

```tsx
import { Modal, Button, Form, Input } from "antd"

{/* 声明式 */}
<Modal
  title="编辑用户"
  open={open}
  onOk={handleOk}
  onCancel={() => setOpen(false)}
  confirmLoading={loading}
  width={600}
  footer={[
    <Button key="cancel" onClick={() => setOpen(false)}>取消</Button>,
    <Button key="submit" type="primary" loading={loading} onClick={handleOk}>确认</Button>,
  ]}
>
  <Form form={form} layout="vertical">
    <Form.Item name="name" label="姓名"><Input /></Form.Item>
  </Form>
</Modal>

{/* 命令式（无需维护 open 状态） */}
import { App } from "antd"

function MyComponent() {
  const { modal } = App.useApp()

  function confirm() {
    modal.confirm({
      title: "确认删除？",
      content: "删除后数据无法恢复",
      okText: "确认",
      okType: "danger",
      cancelText: "取消",
      onOk: async () => {
        await deleteItem()
      },
    })
  }
}
```

### Drawer 抽屉

```tsx
import { Drawer, Button, Form } from "antd"

<Drawer
  title="新建用户"
  placement="right"
  width={500}
  open={open}
  onClose={() => setOpen(false)}
  extra={
    <Space>
      <Button onClick={() => setOpen(false)}>取消</Button>
      <Button type="primary" onClick={handleSubmit}>提交</Button>
    </Space>
  }
>
  <Form layout="vertical">
    {/* 表单内容 */}
  </Form>
</Drawer>
```

### Message 全局消息

```tsx
import { App } from "antd"

// 推荐：通过 App 上下文使用（支持主题）
function MyComponent() {
  const { message } = App.useApp()

  function handleClick() {
    message.success("操作成功")
    message.error("操作失败")
    message.warning("注意事项")
    message.info("提示信息")
    message.loading("加载中...", 0)  // 0 表示不自动关闭

    // Promise 用法
    const hide = message.loading("提交中...", 0)
    await submitForm()
    hide()
    message.success("提交成功")
  }
}

// App 包裹
function App() {
  return (
    <ConfigProvider>
      <AntdApp>  {/* 必须包裹才能使用 useApp() */}
        <RouterProvider router={router} />
      </AntdApp>
    </ConfigProvider>
  )
}
```

### Notification 通知提醒

```tsx
import { App } from "antd"

function MyComponent() {
  const { notification } = App.useApp()

  function openNotification() {
    notification.success({
      message: "操作成功",
      description: "用户信息已更新，变更将在下次登录后生效。",
      placement: "topRight",   // topLeft | topRight | bottomLeft | bottomRight
      duration: 4.5,
      btn: (
        <Button type="primary" size="small" onClick={() => notification.destroy()}>
          确认
        </Button>
      ),
    })

    notification.error({ message: "请求失败", description: "网络连接超时，请稍后重试" })
  }
}
```

### Popconfirm 气泡确认

```tsx
import { Popconfirm, Button } from "antd"

<Popconfirm
  title="确认删除"
  description="此操作不可撤销，确定要删除吗？"
  onConfirm={() => handleDelete(id)}
  onCancel={() => console.log("取消")}
  okText="确认"
  cancelText="取消"
  okType="danger"
>
  <Button danger>删除</Button>
</Popconfirm>
```

### Spin 加载中

```tsx
import { Spin } from "antd"

{/* 整页加载 */}
<Spin spinning={loading} fullscreen />

{/* 包裹内容 */}
<Spin spinning={loading} tip="加载中...">
  <div style={{ padding: 50 }}>需要加载的内容</div>
</Spin>

{/* 自定义指示符 */}
<Spin indicator={<LoadingOutlined style={{ fontSize: 24 }} spin />} />
```

### Result 结果

```tsx
import { Result, Button } from "antd"

<Result
  status="success"
  title="提交成功！"
  subTitle="订单号：2024123456，预计 3 天内完成"
  extra={[
    <Button type="primary" key="home">回到首页</Button>,
    <Button key="order">查看订单</Button>,
  ]}
/>

<Result status="404" title="404" subTitle="您访问的页面不存在" extra={<Button type="primary">返回首页</Button>} />
<Result status="500" title="500" subTitle="服务器错误" extra={<Button type="primary">刷新页面</Button>} />
<Result status="403" title="403" subTitle="无访问权限" />
```

---

## 导航组件

### Menu 菜单

```tsx
import { Menu } from "antd"
import type { MenuProps } from "antd"
import { useNavigate, useLocation } from "react-router"

type MenuItem = Required<MenuProps>["items"][number]

const items: MenuItem[] = [
  { key: "/",          icon: <DashboardOutlined />, label: "仪表盘" },
  { key: "/users",     icon: <UserOutlined />,      label: "用户管理" },
  { key: "/orders",   icon: <ShoppingOutlined />,  label: "订单管理" },
  {
    key: "setting",
    icon: <SettingOutlined />,
    label: "系统设置",
    children: [
      { key: "/settings/basic",  label: "基本设置" },
      { key: "/settings/roles",  label: "角色权限" },
      { key: "/settings/logs",   label: "操作日志" },
    ],
  },
]

function SideMenu() {
  const navigate = useNavigate()
  const location = useLocation()

  return (
    <Menu
      mode="inline"
      selectedKeys={[location.pathname]}
      defaultOpenKeys={["setting"]}
      items={items}
      onClick={({ key }) => navigate(key)}
    />
  )
}
```

### Breadcrumb 面包屑

```tsx
import { Breadcrumb } from "antd"

<Breadcrumb
  items={[
    { title: <HomeOutlined />, href: "/" },
    { title: "用户管理", href: "/users" },
    { title: "张三" },
  ]}
/>

{/* 搭配路由自动生成 */}
<Breadcrumb
  items={[
    { title: "首页", onClick: () => navigate("/") },
    ...breadcrumbs.map((b) => ({
      title: b.label,
      onClick: b.path ? () => navigate(b.path) : undefined,
    })),
  ]}
/>
```

### Tabs 标签页

```tsx
import { Tabs } from "antd"

<Tabs
  defaultActiveKey="1"
  type="card"   // line（默认）| card | editable-card
  items={[
    { key: "1", label: "基本信息", children: <BasicInfo /> },
    { key: "2", label: "安全设置", children: <SecuritySettings /> },
    { key: "3", label: "操作日志", children: <OperationLog />, disabled: !isAdmin },
  ]}
  onChange={(key) => console.log(key)}
/>
```

### Steps 步骤条

```tsx
import { Steps } from "antd"

<Steps
  current={1}
  items={[
    { title: "填写信息", description: "基本资料" },
    { title: "身份认证", description: "实名认证" },
    { title: "完成注册", description: "账号已激活" },
  ]}
/>

{/* 竖向步骤条（带内容） */}
<Steps
  direction="vertical"
  current={current}
  items={steps.map((s) => ({
    title: s.title,
    description: current >= s.index ? <s.content /> : null,
  }))}
/>
```

---

## 实用 Hook

### useApp

```tsx
import { App } from "antd"

// 在根组件包裹 App
function Root() {
  return <App><RouterProvider router={router} /></App>
}

// 在子组件中使用
function MyPage() {
  const { message, modal, notification } = App.useApp()

  async function handleDelete() {
    await modal.confirm({ title: "确认删除？" })
    await deleteItem()
    message.success("删除成功")
  }
}
```

### 表格常用封装

```tsx
// 封装带分页、搜索、增删改查的通用 Hook
function useTable<T>(fetchFn: (params: any) => Promise<{ list: T[]; total: number }>) {
  const [data, setData] = useState<T[]>([])
  const [loading, setLoading] = useState(false)
  const [pagination, setPagination] = useState({ current: 1, pageSize: 10, total: 0 })

  async function fetch(params = {}) {
    setLoading(true)
    try {
      const res = await fetchFn({ ...params, page: pagination.current, pageSize: pagination.pageSize })
      setData(res.list)
      setPagination(p => ({ ...p, total: res.total }))
    } finally {
      setLoading(false)
    }
  }

  useEffect(() => { fetch() }, [pagination.current, pagination.pageSize])

  const tableProps = {
    dataSource: data,
    loading,
    pagination: {
      ...pagination,
      showSizeChanger: true,
      showTotal: (total: number) => `共 ${total} 条`,
      onChange: (page: number, pageSize: number) => setPagination(p => ({ ...p, current: page, pageSize })),
    },
  }

  return { tableProps, fetch, loading }
}

// 使用
function UserPage() {
  const { tableProps, fetch } = useTable(fetchUsers)

  return (
    <>
      <Button onClick={fetch}>刷新</Button>
      <Table {...tableProps} columns={columns} rowKey="id" />
    </>
  )
}
```

---

## 常用组件速查

| 类别 | 组件 | 用途 |
|------|------|------|
| **布局** | Layout / Sider | 页面整体布局 |
| | Row / Col | 24 列栅格 |
| | Space | 子元素间距 |
| | Divider | 分隔线 |
| **通用** | Button | 按钮 |
| | Icon | 图标（@ant-design/icons） |
| | Typography | 标题/文本/段落 |
| **数据录入** | Form | 表单（含校验） |
| | Input / Input.Password / Input.TextArea | 输入框 |
| | Select | 选择器 |
| | DatePicker / RangePicker | 日期/范围选择 |
| | InputNumber | 数字输入 |
| | Checkbox / Radio | 复选框/单选 |
| | Switch | 开关 |
| | Slider | 滑块 |
| | Upload | 文件上传 |
| | Transfer | 穿梭框 |
| | TreeSelect | 树形选择 |
| **数据展示** | Table | 表格（排序/筛选/分页） |
| | List | 列表 |
| | Descriptions | 描述列表 |
| | Statistic | 统计数值 |
| | Tag | 标签 |
| | Badge | 徽标 |
| | Avatar | 头像 |
| | Image | 图片（含预览） |
| | Tree | 树形展示 |
| | Timeline | 时间轴 |
| | Progress | 进度条 |
| | Calendar | 日历 |
| | Card | 卡片 |
| | Collapse | 折叠面板 |
| | Carousel | 走马灯 |
| | Popover | 气泡卡片 |
| | Tooltip | 文字提示 |
| **反馈** | Modal | 对话框 |
| | Drawer | 抽屉 |
| | Message | 全局消息 |
| | Notification | 通知提醒 |
| | Popconfirm | 气泡确认 |
| | Spin | 加载中 |
| | Skeleton | 骨架屏 |
| | Result | 结果页 |
| | Alert | 警告提示 |
| | Empty | 空状态 |
| **导航** | Menu | 菜单 |
| | Breadcrumb | 面包屑 |
| | Tabs | 标签页 |
| | Steps | 步骤条 |
| | Pagination | 分页 |
| | Anchor | 锚点 |
