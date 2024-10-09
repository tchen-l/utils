# 日常开发使用到的函数

## 数组

### 移动数组项（修改原数组）
```
const arrayMoveMutable = <T = any>(array: T[], fromIndex: number, toIndex: number) => {
  const startIndex = fromIndex < 0 ? array.length + fromIndex : fromIndex;

  if (startIndex >= 0 && startIndex < array.length) {
    const endIndex = toIndex < 0 ? array.length + toIndex : toIndex;

    const [item] = array.splice(fromIndex, 1);
    array.splice(endIndex, 0, item);
  }
};
```
### 移动数组项（不修改原数组）
```
const arrayMoveImmutable = <T = any>(array: T[], fromIndex: number, toIndex: number) => {
  const newArray = [...array];
  arrayMoveMutable(newArray, fromIndex, toIndex);
  return newArray;
}
```

### 树操作
```
import { isArray, omit } from 'lodash';

type CompareType<T> = (n: T) => boolean;

type RecordWithChildren<T> = { children?: T[] };

function enhanceTree<T extends RecordWithChildren<T>, R extends { children?: R[] }>(
  tree: T | undefined,
  enhance: (node: T, { parent, level }: { level: number; parent?: R | null }) => R,
  info?: { level: number; parent?: R | null }
): R | undefined;

function enhanceTree<T extends RecordWithChildren<T>, R extends { children?: R[] }>(
  tree: T[] | undefined,
  enhance: (node: T, { parent, level }: { level: number; parent?: R | null }) => R,
  info?: { level: number; parent?: R | null }
): R[] | undefined;

function enhanceTree<T extends RecordWithChildren<T>, R extends { children?: R[] }>(
  tree: T | T[] | undefined,
  enhance: (node: T, { parent, level }: { level: number; parent?: R | null }) => R,
  info?: { level: number; parent?: R | null }
): R | R[] | undefined {
  const { level = 1, parent = null } = info || { level: 1, parent: null };

  if (isArray(tree)) {
    return tree.map((node) =>
      enhanceTree(node, enhance, {
        level,
        parent
      })
    ) as R[];
  }
  if (tree) {
    const enhancedObj = enhance(tree, { level, parent });
    if (Array.isArray(tree.children)) {
      enhancedObj.children = enhanceTree<T, R>(tree.children, enhance, {
        level: level + 1,
        parent: enhancedObj
      }) as R[];
    }
    return enhancedObj as R;
  }

  return undefined;
}

function find<T extends RecordWithChildren<T>>(
  tree: T | undefined,
  compare: CompareType<T>
): T | null | undefined;

function find<T extends RecordWithChildren<T>>(
  tree: T[] | undefined,
  compare: CompareType<T>
): { index: number; node: T; siblings: T[] } | null | undefined;

function find<T extends RecordWithChildren<T>>(
  tree: T | T[],
  compare: CompareType<T>
): { index: number; node: T; siblings: T[] } | T | undefined | null {
  let found: { index: number; node: T; siblings: T[] } | T | null | undefined = null;
  if (!tree) {
    return found;
  }

  if (Array.isArray(tree)) {
    for (let i = 0; i < tree.length; i += 1) {
      const node = tree[i];
      if (compare(node)) {
        found ??= {
          index: i,
          node,
          siblings: tree
        };
      }
      if ((node?.children as T[])?.length > 0) {
        found ??= find(node.children as T[], compare);
      }
    }
  } else if (compare(tree)) {
    found ??= tree;
  } else if ((tree?.children as T[])?.length > 0) {
    found ??= find(tree.children as T[], compare);
  }
  return found;
}

function mutate<T extends RecordWithChildren<T>, R>(
  tree: T[],
  compare: CompareType<T>,
  mutation: (data: { index: number; node: T; siblings: T[] }) => void
) {
  const treeCopy = JSON.parse(JSON.stringify(tree)) as T[];
  const found = find(treeCopy, compare);
  if (found) {
    mutation(found);
  }
  return treeCopy as R;
}

function insert<T extends RecordWithChildren<T>>(
  tree: T[],
  compare: CompareType<T>,
  target: T,
  offset: number
) {
  return mutate<T, T[]>(tree, compare, (found) => {
    found.siblings.splice(found.index + offset, 0, target);
  });
}

function prepend<T extends RecordWithChildren<T>>(
  tree: T[],
  compare: CompareType<T>,
  target: T
) {
  return insert<T>(tree, compare, target, 0);
}

function append<T extends RecordWithChildren<T>>(
  tree: T[],
  compare: CompareType<T>,
  target: T
) {
  return insert<T>(tree, compare, target, 1);
}

function remove<T extends RecordWithChildren<T>>(tree: T[], compare: CompareType<T>) {
  return mutate<T, T[]>(tree, compare, (found) => {
    found.siblings.splice(found.index, 1);
  });
}
```

## 数字

### 保留小数
```
/**
 * 安全的处理四舍五入,保留 digits 位小数
 * 在处理数值保留小数位的时候,鉴于后端会返回各种奇怪的数据类型,需要进行类型判断,以避免页面各种 NaN 或报错
 */
const keepDecimalsOrPercentage = (
  value?: unknown,
  {
    digits = 2,
    defaultValue = '-',
    percentage = false
  }: {
    /** 无法处理时返回的默认值 */
    defaultValue?: number | string;
    /** 保留小数位数 */
    digits?: number;
    /**  结果是否处理成百分比(函数内部自动 * 100) */
    percentage?: boolean;
  } = {}
) => {
  const valueType = typeof value;

  // 非 'string', 'number' 类型直接返回 defaultValue
  if (!['string', 'number'].includes(valueType)) return defaultValue;

  const transformedValue = Number(value);

  // 转成 Number 后是 NaN 直接返回 defaultValue
  if (Number.isNaN(transformedValue)) return defaultValue;

  // 处理四舍五入,保留小数
  return percentage
    ? `${(transformedValue * 100).toFixed(digits)}%`
    : transformedValue.toFixed(digits);
};
```

## 字符串

### 文本前置省略或者后置省略
```
const ellipsisText = ({
  type = 'after',
  text,
  limit
}: {
  limit: number;
  text: string | undefined;
  type?: 'before' | 'after';
}) => {
  if (!text || text?.length <= limit) {
    return {
      ellipsis: false,
      text
    };
  }

  if (type === 'after') {
    return {
      ellipsis: true,
      text: `${text.slice(0, limit)}...`
    };
  }

  return {
    ellipsis: true,
    text: `...${text.slice(limit)}`
  };
};
```
### 随机生成指定长度字符串
```
 const randomString = (length: number) => {
  const templateStr = 'ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz2345678';
  const tLen = templateStr.length;
  let str = '';

  for (let i = 0; i < length; i++) {
    str += templateStr.charAt(Math.floor(Math.random() * tLen));
  }

  return str;
};
```

## 其他

### 判读是否为空
```
const isNullOrUndefined = (val?: unknown, extraEmptyUnits?: any[]) =>
  [undefined, null, ...(extraEmptyUnits || [])].includes(val)
```

### url 压缩
```
import lz from 'lz-string';

const JSON_START = /^\[|^\{(?!\{)/;
const JSON_ENDS = {
  '[': /]$/,
  '{': /}$/
};

const isJsonLike = (str: string) => {
  const jsonStart = str.match(JSON_START);
  return jsonStart && JSON_ENDS[jsonStart[0]].test(str);
}

const parseUrlObj = (urlObjStr?: string) => {
  return urlObjStr
    ? JSON.parse(
        isJsonLike(urlObjStr) ? urlObjStr : lz.decompressFromEncodedURIComponent(urlObjStr)
      )
    : urlObjStr;
}

const stringifyObjUrl = (obj: any) => {
  const objStr = JSON.stringify(obj);
  return objStr && lz.compressToEncodedURIComponent(objStr);
}

```
