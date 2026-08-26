# Documentation search

## Documentation Domain Knowledge
You can search and read the official platform documentation dynamically.

### Documentation Directory Structure
Format: - {relative_path}: {page_title}

* /
  - /: /
* /best_practice
  - /best_practice: Zion 最佳实践案例总览
  - /best_practice/advanced_code/code_component_cursor: 使用 Cursor 辅助开发 Zion 代码组件
  - /best_practice/ai_applications/ai_content_classifier: AI内容自动分类
  - /best_practice/ai_applications/ai_copy_reviewer: 如何使用 AI 实现文案风格审查？
  - /best_practice/ai_applications/ai_faq_matching: 如何搭建 AI 知识库智能助手？
  - /best_practice/ai_applications/ai_image_describer: 如何构建 AI 图片自动描述功能？
  - /best_practice/ai_applications/ai_predictive_text_input: 如何实现 AI 输入内容预补全？
  - /best_practice/ai_applications/ai_product_image_generation: 如何实现 AI 产品图片生成功能?
  - /best_practice/ai_applications/ai_resume_parser: AI简历信息提取
  - /best_practice/ai_applications/ai_smart_tagger: AI智能标签标注
  - /best_practice/ai_applications/ai_spam_detector: 如何使用 AI 自动识别并过滤垃圾信息？
  - /best_practice/ai_applications/ai_summarizer_and_translator: 如何实现 AI 文本摘要与翻译功能？
  - /best_practice/ai_applications/ai_travel_planner: AI旅行规划
  - /best_practice/ai_applications/ai_travel_planner/ai_travel_planner_1: AI旅行规划：API与AI Agent配置
  - /best_practice/ai_applications/ai_travel_planner/ai_travel_planner_2: AI旅行规划：数据循环存储
  - /best_practice/ai_applications/ai_travel_planner/ai_travel_planner_3: AI 旅行规划：未读提示
  - /best_practice/automations/auto_downgrade_membership: 会员到期自动降级
  - /best_practice/automations/order_status_auto_updater: 订单超时自动取消
  - /best_practice/feature_modules/basic_star_rating: 星级评分功能最佳实践
  - /best_practice/feature_modules/clean_up_oss_images_while_deleting_data: 如何在删除数据的同时清理 OSS 中的图片？
  - /best_practice/feature_modules/content_visibility_control: 内容可见性控制
  - /best_practice/feature_modules/daily_claim_limits: 每日限额领取实现方案对比
  - /best_practice/feature_modules/daily_claim_limits/state_counter: 每日限额领取（状态记录）
  - /best_practice/feature_modules/daily_claim_limits/unique_constraint: 每日限额领取（约束）
  - /best_practice/feature_modules/data_entry_validation: 数据录入校验最佳实践
  - /best_practice/feature_modules/form_drafts: 表单草稿功能实现方案对比
  - /best_practice/feature_modules/form_drafts/click_to_save: 表单草稿（点击保存）
  - /best_practice/feature_modules/form_drafts/realtime_save: 表单草稿（实时保存）
  - /best_practice/feature_modules/implement_referral_code_generation_and_attribution: 自动生成并校验用户专属邀请码?
  - /best_practice/feature_modules/like: 点赞功能最佳实践
  - /best_practice/feature_modules/login_register: 网页端登录与注册最佳实践
  - /best_practice/feature_modules/multi_file_upload_and_management: How to Upload and Manage Multiple Files?
  - /best_practice/feature_modules/nearby_store_view: 附近门店查看最佳实践
  - /best_practice/feature_modules/nested_list_seat_booking: 如何利用嵌套列表实现选座功能？
  - /best_practice/feature_modules/order_inventory_deduction: 订单库存扣减
  - /best_practice/feature_modules/password_strength_validation: 密码强度校验（前端逻辑实现）
  - /best_practice/feature_modules/sidebar: 侧边栏功能最佳实践
  - /best_practice/feature_modules/web_verification_code_countdown: Web端验证码倒计时最佳实践
  - /best_practice/integrations/integrate_express_100: 对接快递100最佳实践
  - /best_practice/integrations/official_account_to_wechat_mini_program: 公众号跳转小程序最佳实践
  - /best_practice/integrations/wechat_mini_program_ad: 微信小程序广告最佳实践
  - /best_practice/integrations/wechat_mini_program_to_official_account: 小程序跳转公众号最佳实践
  - /best_practice/integrations/wechat_official_accounts_send_message: 微信服务号消息通知最佳实践
* /changelog
  - /changelog: 产品更新日志
* /docs
  - /docs: 产品简介与阅读指引
  - /docs/actions: 行为配置
  - /docs/actions/concept/interaction_model: 交互模型
  - /docs/actions/guide/ai_integration: 搭建 AI Agent
  - /docs/actions/guide/api_integration: 集成第三方 API
  - /docs/actions/guide/building_action_flows: 搭建行为流
  - /docs/actions/guide/frontend_interactions: 配置前端行为
  - /docs/actions/guide/payment/payment_airwallex: Airwallex 支付
  - /docs/actions/guide/payment/payment_alipay: 支付宝支付
  - /docs/actions/guide/payment/payment_overview: 支付功能概述
  - /docs/actions/guide/payment/payment_wechat_miniprogram: 微信支付（小程序端）
  - /docs/actions/guide/payment/payment_wechat_web: 微信支付（Web 端）
  - /docs/actions/guide/sso_configuration: 使用第三方登录
  - /docs/actions/reference: 参考手册
  - /docs/actions/reference/actionflow_node_list: 行为流节点
  - /docs/actions/reference/app_page: 客户端和页面行为
  - /docs/actions/reference/component_operations: 组件行为
  - /docs/actions/reference/condition: 条件
  - /docs/actions/reference/database_operation: 数据库操作
  - /docs/actions/reference/for_each: 循环
  - /docs/actions/reference/navigation: 导航行为
  - /docs/actions/reference/show_toast: 提示和弹窗
  - /docs/actions/reference/system: 系统
  - /docs/actions/reference/trigger_list: 触发器
  - /docs/actions/reference/user_event_collection: 用户行为
  - /docs/actions/reference/wechat_features: 微信功能
  - /docs/billing_commercial: 项目与计费
  - /docs/billing_commercial/commission_rule: 推广者计划
  - /docs/billing_commercial/my_wallet: 我的钱包
  - /docs/billing_commercial/project_team_collaboration: 项目管理与协作
  - /docs/billing_commercial/resource_management: 管理项目资源
  - /docs/billing_commercial/upgrade_plan: 升级项目版本
  - /docs/data: 数据
  - /docs/data/concept/data_model: 数据模型与流转
  - /docs/data/concept/database_relation_model: 关系型数据库思维
  - /docs/data/concept/variables_and_parameters: 变量和参数
  - /docs/data/guide/bird_eye_view: 使用数据鸟瞰图
  - /docs/data/guide/conditional_data: 配置条件数据
  - /docs/data/guide/data_management: 数据管理
  - /docs/data/guide/database_configuration: 数据模型配置
  - /docs/data/guide/databinding_and_query: 获取数据源与数据绑定
  - /docs/data/guide/import_and_export: 数据导入与导出
  - /docs/data/guide/resource_manager: 多媒体资源管理
  - /docs/data/guide/secret_management: 密钥管理
  - /docs/data/guide/variable_parameter_usage: 使用页面变量和页面参数
  - /docs/data/guide/vector_data: 向量存储与排序
  - /docs/data/reference/data_types: 数据类型
  - /docs/data/reference/formula_dictionary: 公式参考
  - /docs/design: UI 搭建
  - /docs/design/concept/layout_concepts: 布局系统
  - /docs/design/concept/ui_organization_model: 理解 Zion 的 UI 组织方式
  - /docs/design/guide/studio_basics: 搭建第一个页面
  - /docs/design/guide/using_custom_components: 使用自定义组件
  - /docs/design/reference/breakpoints: 断点规格
  - /docs/design/reference/design_panel: 设计面板
  - /docs/design/reference/display_components: 展示类组件
  - /docs/design/reference/input_components: 输入类组件
  - /docs/design/reference/layout_components: 布局类组件
  - /docs/design/reference/other_components: 其他类组件
  - /docs/design/reference/page_modal: 页面与弹窗
  - /docs/design/reference/ui_shortcuts: UI 快捷键
  - /docs/developers: 开发者集成
  - /docs/developers/code_component: 开发代码组件
  - /docs/developers/headless: Headless · Zion BaaS
  - /docs/developers/run_code: 运行代码节点开发
  - /docs/developers/runtime_api: Runtime API 参考
  - /docs/publish_operate: 发布与运维
  - /docs/publish_operate/app_deployment: 发布应用
  - /docs/publish_operate/custom_domain: 使用自定义域名
  - /docs/publish_operate/log_service: 查看运行日志
  - /docs/publish_operate/mirror: 实时预览
  - /docs/publish_operate/multiple_frontends: 搭建多客户端应用
  - /docs/publish_operate/permissions: 管理应用权限
  - /docs/publish_operate/reference/error_dictionary: 错误参考
  - /docs/publish_operate/reference/rendering_modes: 页面渲染模式
  - /docs/publish_operate/seo: Web 应用 SEO
  - /docs/publish_operate/troubleshooting_guide: 排查应用问题
  - /docs/starts/editor_overview: 认识 Zion 编辑器
  - /docs/starts/glossary: 产品术语表
  - /docs/starts/hello_world: 5分钟 待办事项应用 实战
  - /docs/starts/mental_models: 建立心智模型
  - /docs/starts/methodology: 软件设计方法论
* /templates
  - /templates: 模板使用教程
  - /templates/content/blog_template_tutorial: 博客模板使用教程
  - /templates/content/knowledge_paid_course_template_tutorial: 知识付费课程模板使用教程
  - /templates/content/smart_knowledge_base_tutorial: AI知识库模板使用教程
  - /templates/ecommerce/ecommerce_backend_tutorial: 电商模板后台使用教程
  - /templates/ecommerce/ecommerce_template_tutorial: 电商模板使用教程
  - /templates/services/home_services_template_tutorial: 家政服务模板使用教程
  - /templates/services/official_wechat_mini_program_template_tutorial: Zion 官方小程序模板使用教程
  - /templates/utilities/ai_assistant_template_tutorial: AI 助理模板使用教程
  - /templates/utilities/illustrated_guide_template_tutorial: 图鉴模板使用教程
  - /templates/utilities/onboarding_template_tutorial: 新手引导项目使用教程

### Guidelines
1. When the user asks "how-to" questions, or requests guides, call docs.search first.
2. If a relevant page path is clearly visible in the directory structure above, you may directly call docs.get_page with that path without searching first.
3. Call docs.get_page to read the actual page content to extract accurate steps and facts.
4. Ground all explanations in the loaded documentation. Do not guess or fabricate.

## How to drive it (CLI only)

Read-only — no schema session needed.

```bash
npx -y zion-mcp@2.7.0 docs search --query "how to configure wechat pay / alipay payments"
npx -y zion-mcp@2.7.0 docs get-page --path "/03_data/01_database_basics"
```

`docs search` returns `{ path, title, url }` ranked by relevance; `url` is a public HTTPS link you can cite. Pass a returned `path` to `docs get-page` to read the full markdown. Search before answering how-to questions, and ground every claim in the retrieved page.
