---
title: Low Code 之路（一）
date: 2020/12/15
cover: https://ss3.bdstatic.com/70cFv8Sh_Q1YnxGkpoWK1HF6hhy/it/u=962309492,3418279211&fm=26&gp=0.jpg
categories:
- LowCode
tags: 
- form

---
# Low Code 之路（一）

## 背景

20年要做一些新的事情，终于可以脱离繁复的业务逻辑，主导一些更加有趣的事情。

先来聊下我之前负责的业务流程是什么样的，即传统的业务是如何进行的。

![image-20210327155304930](https://technologybook.tech/assets/img/image-20210327155304930.png)

当然这一套系统还包含其他的一些细节包括错误通知，处理流程等复杂的内容，但其核心就如图示

这种方案的问题

- 每个国家对应一套form再对应一套代码，维护成本高
- 服务端返回也是一一对应
- 同一，因为使领馆常有变化，维护成本更高，几乎不可复用。
- 代码独立性太高，故障率高

所以今年我主要投入到对这个流程的优化的工作中来， 主要的工作围绕着 建立一个通用表单， 建立一个通用填写脚本两个方面来进行，目标即使，就需要相应的配置文件，任何人都可以基于配置文件进行配置，简单的修改配置文件就能生成新的表单，和规定相应的爬取流程来对内容进行获取和填充。

![image-20210327160340220](https://technologybook.tech/assets/img/image-20210327160340220.png)



### 调研阶段

#### 市面上的竞品

***Formily***

> Formily 解决方案的本质是构造了一个 Observable Form Graph，在这个 Form Graph 中，我们抽象了整个表单领域模型，同时这个模型又是一个无限循环状态机。

读了下代码主要是基于RX的Observeable Form Graph状态机，基本是通过component type找到render的内容，然后通过一个基于Rx 的 所谓Form graph来维护全局的状态，读JSON 来render Form视图然后通过key来建立field relation，然后维护全局状态，主要的工作在对Form 数据结构及数据更新算法和一些性能优势，搞出一套updater tree 和 path match性能不错，而且是经过大量用户验证的，包括阿里内部验证的可靠的库。

***Amis***

> amis 是一个低代码前端框架，它使用 JSON 配置来生成页面，可以减少页面开发工作量，极大提升效率。

初始化接口，数据链的设计，更偏向业务一些，包括表达式，联动， renderer都让人感到这是一个很reactive的库，简单易用，源代码没有看，想想大概差不多。

> 渲染过程就是根据节点 path 信息，跟组件池中的组件 `test` (检测) 信息做匹配，如果命中，则把当前节点转给对应组件渲染，节点中其他属性将作为目标组件的 props。需要注意的是，如果是容器组件，比如以上例子中的 `page` 组件，从 props 中拿到的 `body` 是一个子节点，由于节点类型是不固定，由使用者决定，所以不能直接完成渲染，所以交给属性中下发的 `render` 方法去完成渲染，`{render('body', body)}`，他的工作就是拿子节点的 path 信息去组件池里面找到对应的渲染器，然后交给对应组件去完成渲染

***FormRender***

> 通过 JSON Schema 生成标准 Form，常用于自定义搭建配置界面生成

它提供一个 表单设计器，和基于JSON的formcreate，文档写的不是很好。。，代码里也是用global context维护状态，通过eval实现表达式，比较灵活，支持几种标准类型，通过schema type类型来确定渲染内容这个我不是很中意，也支持自定义type component，但是目前看bug比较多，更新策略也是全量更新，没有优化，性能差一些。

还有一些诸如formcreator等等库，方案都大同小异，同一上的调研确定了几点

- 基于JSON Schema 的配置文件。
- 提供 接口接入标准
- 接口字段到schema的映射语法
- 支持template
- 更新粒度以path为依准

![image-20210327163922088](https://technologybook.tech/assets/img/image-20210327163922088.png)





一下是部分实现

store 是基于 Rx 的数据控制中心

通过只有有效更新才能设置form

```typescript
export default class Manage<T extends HashObj> implements IManage<T> {
    public $form: Subject<T> = new Subject<T>();
    private _storeForm: T;
    public formData: T;
    private static instance: Manage<any>;
    private readonly isFreeze: boolean = false;
    private haveSetDefault = false;
    public validations: Map<string, IValidation> = new Map<string, IValidation>();
    private engine = new TemplateEngine();

    get storeForm() {
        return this._storeForm;
    }

    set storeForm(data) {
        if (!this.isFreeze) {
            this._storeForm = { ...data };
        }
    }

    get data() {
        return this.formData;
    }
		.....
```



更新粒度为path

```typescript
    public notifyByPath = (path: string, changed: HashObj, other?: HashObj): void => {
        console.log('I\'m in ', path, this.formData, this.validations);
        const originPath = other!.path;
        const validation = this.validations.get(originPath);
        const newData = produce(this.formData, draft => {
            set(draft, path, changed);
            if (validation && !isEmptyArray(validation.rules)) {
                const errors = validation.rules.map((validatorName: string) => {
                    let defaultValidatorFunc = Validators?.[validatorName];
                    let validatorFunc = this.actions?.[validatorName];
                    if (typeof validatorFunc === 'function') {
                        return validatorFunc(changed);
                    }
                    if (typeof defaultValidatorFunc === 'function') {
                        return defaultValidatorFunc?.(changed);
                    }

                    throw new Error('invalid validation in' + originPath);
                }).filter(r => r !== ValidationResult.PASS);
                const currentComp = draft.components.find(comp => comp.path === originPath)?.validation;
                if (currentComp) {
                    currentComp.errors = errors;
                    set(draft, originPath + 'components', currentComp);
                }
            }
        });

        this.notify(newData);
    };
```



整体的状态维护也是基于Context。

这样一个基础的状态管里就完成了，

接下来需要根据type渲染

```typescript
public buildDataTree(dataStruct: HashObj, components: TAllComponents[]): TAllComponents[] | void {
        if (!Array.isArray(components)) {
            throw new TypeError('ComponentTree->buildDataTree: Wrong Type .Params Must Be Array');
        }
        const newComponents: TAllComponents[] = [];
        for (let i = 0, len = components.length; i < len; i++) {
            const componentItem = components[i];
            if (isUndefined(componentItem)) {
                throw new Error(`buildDataTree: Component Invalid`);
            }

            const { path, type } = componentItem || {};
            const currentVal = get(dataStruct, path);

            if (isUndefined(currentVal)) {
                throw new Error(
                    `buildDataTree: current: wrong path ${path}, components should have corresponding component`,
                );
            }

            if (!isObject(currentVal) || type === FormItemType.CUSTOM) {
                const produceItem = produce(componentItem, draft => {
                    set(draft, 'value', currentVal);
                    this.buildComponentTree(draft);
                });
                newComponents.push(produceItem);
            }
        }
        return newComponents;
    }

    public buildComponentTree(component: TAllComponents, componentConfig?: TComponentConfig) {
        const { type, typeName } = component;
        let componentType: FormItemType | string = type;
        if (type === FormItemType.CUSTOM) {
            if (!typeName) {
                console.error('custom must have typeName');
            }
            componentType = typeName;
        }
        set(component, '$$component', this.components[componentType]);
    }

    public buildTree(schema: ISchema, componentLib: HashType<ReactElement>): TAllComponents[] | void {
        this.setComponents(componentLib);
        const { data, components } = schema;
        if (!data || !components) {
            throw new TypeError('ComponentTree::buildTree: Data Or Component Is Invalid');
        }
        return this.buildDataTree(data, components);
    }
```



然后是validator

validator本意是要用户自己去确定哪些东西需要被校验，所以并没有写很多的校验方法，仅提供一个基础的校验。

设想是需要将其抽象为一个库，专门维护， compoennt也是一样。

```typescript
import { ValidationResult } from '../../constant';
import { isNullOrUndefined, isEmpty } from '../../utils';


export class Validators {
    public static required = (data: unknown): [ValidationResult.FAIL, string] | ValidationResult.PASS => {
        if (isNullOrUndefined(data) || isEmpty(data)) {
            return [ValidationResult.FAIL, '不能为空！'];
        }
        return ValidationResult.PASS;
    };
}
```

支持模版字符

```typescript
import { HashObj, TTemplateResult } from '../../types/project';
import { get, isFunction, isString, isTotalWord, isUndefined } from "../../utils";
import { safeEval } from './safe-eval';

interface ITemplateEngine<T extends HashObj> {
    execute(tpl: string, data: T, current: any): TTemplateResult|TTemplateResult[];
}
// optimization
interface IExpression<T> {
    analyse(tpl: string, data: T, current: any): TTemplateResult;
}

abstract class TemplateExpression<T extends HashObj = HashObj> implements IExpression<T> {
    static getSymbol(tpl: string) {
        let [anchor, variable] = TemplateEngine.symbolReg.exec(tpl) || [];
        if (!(anchor && variable)) {
            console.error('Input Is Invalid: ' + tpl);
            throw new Error('Input Is Invalid: ' + tpl);
        }
        return [anchor, variable];
    }

    abstract analyse(tpl: string, data: T, current: any): TTemplateResult;
}

class PureExpression extends TemplateExpression {
    analyse(tpl: string): string {
        return tpl;
    }
}

class VariableExpression extends TemplateExpression {
    analyse(tpl: string, data: HashObj, current: any): TTemplateResult {
        let [, code] = TemplateExpression.getSymbol(tpl);
        const action = get(data, ['actions', code]);
        const property = get(data, code);
        return isFunction(action) ? action(current, data) : property;
    }
}

class CalculateExpression extends TemplateExpression {
    analyse(tpl: string, data: HashObj): TTemplateResult {
        let [, code] = TemplateExpression.getSymbol(tpl);
        code = code.replace(TemplateEngine.varReg, (current: string) => {
            let result = get(data, current);
            if (['true', 'false'].includes(current)) {
                return `!!${current}`;
            }
            if (typeof result === 'string') {
                return `"${result}"`;
            }
            return result;
        });
        return safeEval(code) || '';
    }
}

export default class TemplateEngine<T extends HashObj = HashObj> implements ITemplateEngine<T> {
    public static readonly symbolReg: RegExp = /^{{(.+)?}}$/i;
    public static readonly varReg: RegExp = /[A-Za-z.]+(?!["'a-z])/g;

    static isTpl(tpl: string): boolean {
        return TemplateEngine.symbolReg.test(tpl);
    }

    getExpressionHandler(tpl: string, data?: T): TemplateExpression {
        if (!TemplateEngine.isTpl(tpl)) {
            return new PureExpression();
        }

        const [, code] = TemplateExpression.getSymbol(tpl);

        if (isTotalWord(code) && !isUndefined(data)) {
            return new VariableExpression();
        }
        return new CalculateExpression();
    }



    public execute(tpl: string|string[], data: T, current?: any): TTemplateResult|TTemplateResult[] {
        if (isString(tpl)) {
            const handler = this.getExpressionHandler(tpl as string, data);
            return handler.analyse(tpl, data, current);
        }
        return tpl.map(item => {
            const handler = this.getExpressionHandler(item, data);
            return handler.analyse(item, data, current);
        })

    }
}
```



支持一下几种case

```typescript
describe('template-engine', () => {
  it('should execute code', function() {
    const tpl = '{{a}}';
    const mockData = { a: 100 };
    const tplEngine = new TemplateEngine();
    const result = tplEngine.execute(tpl, mockData);
    expect(result).toEqual(100);
  });

  it('safe eval work', () => {
    const tpl = '1+1';
    const result = safeEval(tpl);
    expect(result).toEqual(2);
  });

  it("safe eval is safe", function() {
    const tpl = "1";
    const result = safeEval(tpl);
    expect(result).toBe(1);
    const tpl1 = "onchange";
    const result1 = safeEval(tpl1);
    expect(result1).toBe(undefined)
  });

  it("safe eval throw error", function() {
    const tpl = "asd///"
    expect(() => safeEval(tpl)).toThrow();
  });

  it("safe eval without window", () => {
    const spy = jest.spyOn(utils, 'isUndefined');
    spy.mockReturnValue(true);
    const tpl = "1";
    expect(() => safeEval(tpl)).toThrow();
    spy.mockRestore();
  })

  it('execute expression should work', function() {
    const tpl = '{{1 + 1}}';
    const tplEngine = new TemplateEngine();
    const result = tplEngine.execute(tpl, {});
    expect(result).toEqual(2);
  });

  it('execute expression with variable should work', function() {
    const tpl = '{{a + 1}}';
    const tplEngine = new TemplateEngine();
    const result = tplEngine.execute(tpl, { a: 100 });
    expect(result).toEqual(101);
  });

  it('execute expression with two variable should work', function() {
    const tpl = '{{a + b}}';
    const tplEngine = new TemplateEngine();
    const result = tplEngine.execute(tpl, { a: 100, b: 200 });
    expect(result).toEqual(300);
  });

  it('ternary operator should work', () => {
    const tpl = '{{a > 100 ? 1 : 2}}';
    const mockData = { a: 100 };
    const tplEngine = new TemplateEngine();
    let result = tplEngine.execute(tpl, mockData);
    expect(result).toEqual(2);
    mockData.a = 101;
    result = tplEngine.execute(tpl, mockData);
    expect(result).toEqual(1);
  });

  it('call function should work', () => {
    const tpl = '{{d}}';
    const mockData = { a: { b: { c: 12 } }, actions: {d: (current: any) => current.b.c} };
    const tplEngine = new TemplateEngine();
    let result = tplEngine.execute(tpl, mockData, { b: { c: 12 } });
    expect(result).toEqual(12);
  });

  it('call lang api should work', () => {
    const tpl = '{{[a]}}';
    const mockData = {
    a: 1
  };
    const tplEngine = new TemplateEngine();
    let result = tplEngine.execute(tpl, mockData, { b: { c: 12 } });
    expect(result).toEqual([1]);
  });

  it('expression list will works', () => {
    const tpl = ["{{a}}"];
    const mockData = {
      a: 1
    };
    const tplEngine = new TemplateEngine();
    let result = tplEngine.execute(tpl, mockData, { b: { c: 12 } });
    expect(result).toEqual([1])
  });

  it('compare should work', () => {
    const tpl = '{{a === "a"}}';
    const mockData = {
      a: 'a'
    };
    const tplEngine = new TemplateEngine();
    let result = tplEngine.execute(tpl, mockData, { b: { c: 12 } });
    expect(result).toEqual(true);
  });

  it('compare should work', () => {
    const tpl = '{{a.b === "a"}}';
    const mockData = {
      a: {
        b: "a"
      }
    };
    const tplEngine = new TemplateEngine();
    let result = tplEngine.execute(tpl, mockData);
    expect(result).toEqual(true);
  });
});
```



最后暴露给开发者一些hooks

用来应对不同的场景

```typescript
import { useContext, useEffect, useMemo, useRef } from 'react';
import { IContextParams, SingleContext } from '../core';
import { HashObj, IValidation } from '../types/project';

export function useFormChange<T extends HashObj>(path?: string): [T, (data: T) => void] {
    const formContext = SingleContext.getContext<T>();
    const { state, managerIns } = useContext<IContextParams<T>>(formContext);
    return [state, (changed: HashObj, currentPath?: string) => managerIns.notifyByPath(path || currentPath || '', changed)];
}

export function useManage<T extends HashObj>() {
    const formContext = SingleContext.getContext<T>();
    const { managerIns } = useContext<IContextParams<T>>(formContext);
    return managerIns;
}

export function useValidation<T extends HashObj>(path: string, validation?: IValidation) {
    if (!validation) return;
    const formContext = SingleContext.getContext<T>();
    const { managerIns } = useContext<IContextParams<T>>(formContext);
    useMemo(() => {
        managerIns.registryValidation(path, validation);
    }, []);
}

export function usePrevious<T>(value: T): T|undefined {
    const ref = useRef<T>();
    useEffect(() => {
        ref.current = value;
    }, [value]); // Only re-run if value changes
    return ref.current;
}
```

总结

一个基于JSON Schema 的buildform就完成了，通过全量的接收数据统一了对接口的interface，每次请求通过参数读取config渲染表单，再通过表单渲染实现面向配置渲染页面，稳步线中～😄

