> 低代码平台TypeScript类型系统设计

## 1. 设计挑战与目标

### 核心挑战
- **动态性vs静态类型**：配置是运行时动态的，但需要编译时类型检查
- **扩展性**：支持新组件类型的快速接入
- **类型安全**：保证配置数据的正确性
- **开发体验**：提供良好的IDE智能提示

### 设计目标
```typescript
// 目标1: 类型安全的配置
interface SafeConfig {
  componentType: 'Button' | 'Input' | 'Table';
  props: ComponentProps<typeof componentType>; // 根据组件类型推导props类型
}

// 目标2: 扩展性
type ComponentRegistry = {
  Button: ButtonComponent;
  Input: InputComponent;
  // 新组件可以轻松添加
}

// 目标3: 开发体验
const config: PageConfig = {
  // IDE自动提示所有可用组件和属性
}
```

## 2. 核心类型系统架构

### 组件类型注册机制
```typescript
// 组件基础接口
interface BaseComponent {
  id: string;
  type: string;
  props: Record<string, any>;
  children?: ComponentConfig[];
  events?: EventConfig[];
}

// 组件注册中心
interface ComponentRegistry {
  // 基础组件
  Button: {
    props: {
      text: string;
      type?: 'primary' | 'secondary' | 'danger';
      size?: 'small' | 'medium' | 'large';
      disabled?: boolean;
      onClick?: string; // 事件处理器ID
    };
    slots?: never;
    children?: never;
  };
  
  Input: {
    props: {
      placeholder?: string;
      value?: string;
      type?: 'text' | 'password' | 'email';
      required?: boolean;
      onChange?: string;
    };
    slots?: never;
    children?: never;
  };
  
  // 容器组件
  Container: {
    props: {
      layout?: 'flex' | 'grid';
      padding?: number;
      gap?: number;
    };
    slots?: never;
    children: ComponentConfig[];
  };
  
  // 表格组件
  Table: {
    props: {
      dataSource: string; // 数据源ID
      columns: TableColumn[];
      pagination?: boolean;
      loading?: boolean;
    };
    slots?: {
      header?: ComponentConfig[];
      footer?: ComponentConfig[];
    };
    children?: never;
  };
}

// 表格列定义
interface TableColumn {
  key: string;
  title: string;
  dataIndex: string;
  width?: number;
  render?: string; // 渲染器ID
}
```

### 类型安全的组件配置
```typescript
// 根据组件类型推导配置类型
type ComponentConfig<T extends keyof ComponentRegistry = keyof ComponentRegistry> = {
  id: string;
  type: T;
  props: ComponentRegistry[T]['props'];
  children?: ComponentRegistry[T]['children'];
  slots?: ComponentRegistry[T]['slots'];
  style?: CSSProperties;
  events?: EventConfig[];
}

// 事件配置
interface EventConfig {
  eventType: string;
  handlerType: 'action' | 'navigate' | 'custom';
  handlerConfig: ActionConfig | NavigateConfig | CustomConfig;
}

interface ActionConfig {
  type: 'action';
  actionId: string;
  params?: Record<string, any>;
}

interface NavigateConfig {
  type: 'navigate';
  target: string;
  params?: Record<string, string>;
}

interface CustomConfig {
  type: 'custom';
  code: string;
  async?: boolean;
}
```

### 页面配置类型
```typescript
// 页面完整配置
interface PageConfig {
  id: string;
  title: string;
  description?: string;
  layout: LayoutConfig;
  components: ComponentConfig[];
  dataSources: DataSourceConfig[];
  actions: ActionConfig[];
  styles: StyleConfig;
  meta: PageMeta;
}

// 布局配置
interface LayoutConfig {
  type: 'fixed' | 'responsive';
  width?: number;
  maxWidth?: number;
  padding?: number;
  background?: string;
}

// 数据源配置
interface DataSourceConfig {
  id: string;
  type: 'api' | 'static' | 'computed';
  config: ApiDataSource | StaticDataSource | ComputedDataSource;
}

interface ApiDataSource {
  type: 'api';
  url: string;
  method: 'GET' | 'POST' | 'PUT' | 'DELETE';
  headers?: Record<string, string>;
  params?: Record<string, any>;
  transform?: string; // 数据转换函数
}

interface StaticDataSource {
  type: 'static';
  data: any[];
}

interface ComputedDataSource {
  type: 'computed';
  dependencies: string[]; // 依赖的数据源ID
  compute: string; // 计算函数
}
```

## 3. 高级类型特性

### 条件类型和映射类型
```typescript
// 根据组件类型条件推导
type ComponentPropsType<T extends keyof ComponentRegistry> = 
  ComponentRegistry[T]['props'];

type ComponentChildrenType<T extends keyof ComponentRegistry> = 
  ComponentRegistry[T]['children'];

// 组件配置工厂
type CreateComponentConfig<T extends keyof ComponentRegistry> = {
  type: T;
  props: ComponentPropsType<T>;
} & (ComponentChildrenType<T> extends never 
  ? {} 
  : { children: ComponentChildrenType<T> }
);

// 使用示例
const buttonConfig: CreateComponentConfig<'Button'> = {
  type: 'Button',
  props: {
    text: 'Click me',
    type: 'primary'
  }
  // children 不可用，因为Button不支持children
};

const containerConfig: CreateComponentConfig<'Container'> = {
  type: 'Container',
  props: {
    layout: 'flex',
    padding: 16
  },
  children: [buttonConfig] // children 是必需的
};
```

### 类型守卫和验证
```typescript
// 组件类型守卫
function isComponentType<T extends keyof ComponentRegistry>(
  type: string,
  targetType: T
): type is T {
  return type === targetType;
}

// 配置验证函数
function validateComponentConfig<T extends keyof ComponentRegistry>(
  config: any,
  expectedType: T
): config is ComponentConfig<T> {
  if (!config || typeof config !== 'object') return false;
  if (config.type !== expectedType) return false;
  
  // 根据组件类型进行具体验证
  switch (expectedType) {
    case 'Button':
      return validateButtonProps(config.props);
    case 'Input':
      return validateInputProps(config.props);
    case 'Table':
      return validateTableProps(config.props);
    default:
      return true;
  }
}

function validateButtonProps(props: any): props is ComponentRegistry['Button']['props'] {
  return typeof props.text === 'string' &&
         (props.type === undefined || ['primary', 'secondary', 'danger'].includes(props.type));
}
```

## 4. 运行时类型安全

### 配置序列化与反序列化
```typescript
// 配置序列化
class ConfigSerializer {
  static serialize(config: PageConfig): string {
    return JSON.stringify(config, this.replacer);
  }
  
  static deserialize(json: string): PageConfig {
    const config = JSON.parse(json, this.reviver);
    return this.validateAndMigrate(config);
  }
  
  private static replacer(key: string, value: any): any {
    // 处理函数序列化
    if (typeof value === 'function') {
      return { __type: 'function', code: value.toString() };
    }
    return value;
  }
  
  private static reviver(key: string, value: any): any {
    // 恢复函数
    if (value && value.__type === 'function') {
      return new Function('return ' + value.code)();
    }
    return value;
  }
  
  private static validateAndMigrate(config: any): PageConfig {
    // 配置版本迁移和验证
    if (!this.isValidPageConfig(config)) {
      throw new Error('Invalid page configuration');
    }
    return config;
  }
  
  private static isValidPageConfig(config: any): config is PageConfig {
    return config &&
           typeof config.id === 'string' &&
           typeof config.title === 'string' &&
           Array.isArray(config.components);
  }
}
```

### 动态组件渲染
```typescript
// 组件渲染器
class ComponentRenderer {
  private registry: Map<string, React.ComponentType<any>> = new Map();
  
  registerComponent<T extends keyof ComponentRegistry>(
    type: T,
    component: React.ComponentType<ComponentRegistry[T]['props']>
  ): void {
    this.registry.set(type, component);
  }
  
  renderComponent(config: ComponentConfig): React.ReactElement | null {
    const Component = this.registry.get(config.type);
    if (!Component) {
      console.warn(`Unknown component type: ${config.type}`);
      return null;
    }
    
    // 类型安全的props传递
    const props = this.processProps(config);
    const children = this.renderChildren(config.children);
    
    return React.createElement(Component, props, children);
  }
  
  private processProps(config: ComponentConfig): any {
    // 处理事件绑定
    const props = { ...config.props };
    
    if (config.events) {
      config.events.forEach(event => {
        const handlerName = `on${event.eventType.charAt(0).toUpperCase()}${event.eventType.slice(1)}`;
        props[handlerName] = this.createEventHandler(event);
      });
    }
    
    return props;
  }
  
  private createEventHandler(event: EventConfig): Function {
    return (...args: any[]) => {
      switch (event.handlerConfig.type) {
        case 'action':
          return this.executeAction(event.handlerConfig as ActionConfig, args);
        case 'navigate':
          return this.executeNavigation(event.handlerConfig as NavigateConfig);
        case 'custom':
          return this.executeCustomHandler(event.handlerConfig as CustomConfig, args);
      }
    };
  }
  
  private renderChildren(children?: ComponentConfig[]): React.ReactNode {
    if (!children) return null;
    return children.map(child => this.renderComponent(child));
  }
}
```

## 5. 开发工具支持

### 类型定义生成器
```typescript
// 从组件注册表生成类型定义
class TypeDefinitionGenerator {
  static generateComponentTypes(registry: ComponentRegistry): string {
    let output = '// Auto-generated component types\n\n';
    
    for (const [componentType, definition] of Object.entries(registry)) {
      output += this.generateComponentInterface(componentType, definition);
    }
    
    return output;
  }
  
  private static generateComponentInterface(
    name: string, 
    definition: ComponentRegistry[keyof ComponentRegistry]
  ): string {
    return `
interface ${name}Config extends BaseComponent {
  type: '${name}';
  props: ${JSON.stringify(definition.props, null, 2)};
  ${definition.children ? `children: ComponentConfig[];` : ''}
  ${definition.slots ? `slots: ${JSON.stringify(definition.slots, null, 2)};` : ''}
}
`;
  }
}
```

### 配置编辑器类型支持
```typescript
// 配置编辑器的类型安全接口
interface ConfigEditor {
  // 添加组件
  addComponent<T extends keyof ComponentRegistry>(
    type: T,
    props: ComponentRegistry[T]['props'],
    parentId?: string
  ): ComponentConfig<T>;
  
  // 更新组件属性
  updateComponentProps<T extends keyof ComponentRegistry>(
    componentId: string,
    props: Partial<ComponentRegistry[T]['props']>
  ): void;
  
  // 删除组件
  removeComponent(componentId: string): void;
  
  // 移动组件
  moveComponent(componentId: string, newParentId: string, index: number): void;
}

// 实现类型安全的编辑器
class TypeSafeConfigEditor implements ConfigEditor {
  constructor(private config: PageConfig) {}
  
  addComponent<T extends keyof ComponentRegistry>(
    type: T,
    props: ComponentRegistry[T]['props'],
    parentId?: string
  ): ComponentConfig<T> {
    const component: ComponentConfig<T> = {
      id: this.generateId(),
      type,
      props,
      ...(this.needsChildren(type) && { children: [] }),
      ...(this.hasSlots(type) && { slots: {} })
    };
    
    this.insertComponent(component, parentId);
    return component;
  }
  
  private needsChildren<T extends keyof ComponentRegistry>(type: T): boolean {
    return ComponentRegistry[type]['children'] !== undefined;
  }
  
  private hasSlots<T extends keyof ComponentRegistry>(type: T): boolean {
    return ComponentRegistry[type]['slots'] !== undefined;
  }
}
```

## 6. 性能优化

### 类型缓存和优化
```typescript
// 类型检查缓存
class TypeCheckCache {
  private cache = new Map<string, boolean>();
  
  isValidConfig(config: ComponentConfig): boolean {
    const cacheKey = this.generateCacheKey(config);
    
    if (this.cache.has(cacheKey)) {
      return this.cache.get(cacheKey)!;
    }
    
    const isValid = this.performTypeCheck(config);
    this.cache.set(cacheKey, isValid);
    return isValid;
  }
  
  private generateCacheKey(config: ComponentConfig): string {
    return `${config.type}:${JSON.stringify(config.props)}`;
  }
  
  private performTypeCheck(config: ComponentConfig): boolean {
    // 执行实际的类型检查
    return validateComponentConfig(config, config.type);
  }
  
  clearCache(): void {
    this.cache.clear();
  }
}
```

## 7. 最佳实践总结

### 类型系统设计原则
1. **渐进式类型约束**：从宽松到严格的类型定义
2. **组合优于继承**：使用接口组合而非类继承
3. **类型安全的运行时**：编译时检查 + 运行时验证
4. **开发体验优先**：提供良好的IDE支持和错误提示

### 实际应用建议
```typescript
// 1. 使用严格的TypeScript配置
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true
  }
}

// 2. 提供类型定义文件
declare module '@lowcode/core' {
  export interface ComponentRegistry {
    // 扩展组件类型
  }
}

// 3. 使用类型断言时要谨慎
const config = data as PageConfig; // 避免
const config = validatePageConfig(data) ? data : getDefaultConfig(); // 推荐
```

这样的类型系统设计既保证了类型安全，又保持了低代码平台的灵活性，为开发者提供了良好的开发体验。