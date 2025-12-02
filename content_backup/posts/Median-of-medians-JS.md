---
title: "Median-of-medians无序数组寻找中位数最差O(n)复杂度JS实现"
date: 2019-06-03T00:00:00+08:00
---

在无序数组中寻找中位数，最差复杂度为 O(n). 实现算法为 Median of medians，又叫 BFPRT 算法。
实现原理与复杂度研究：[https://en.wikipedia.org/wiki/Median_of_medians](https://en.wikipedia.org/wiki/Median_of_medians)

贴一版 JS 实现：

```JavaScript
export const selectMedian = (arr, compare) => {
  return selectK(arr, Math.floor(arr.length / 2), compare);
};

export const selectK = (arr, k, compare) => {
  if (!Array.isArray(arr) || arr.length === 0 || arr.length - 1 < k) {
    return;
  }
  if (arr.length === 1) {
    return arr[0];
  }
  let idx = selectIdx(arr, 0, arr.length - 1, k, compare || defaultCompare);
  return arr[idx];
};

const partition = (arr, left, right, pivot, compare) => {
  let temp = arr[pivot];
  arr[pivot] = arr[right];
  arr[right] = temp;
  let track = left;
  for (let i = left; i < right; i++) {
    // if (arr[i] < arr[right]) {
    if (compare(arr[i], arr[right]) === -1) {
      let t = arr[i];
      arr[i] = arr[track];
      arr[track] = t;
      track++;
    }
  }
  temp = arr[track];
  arr[track] = arr[right];
  arr[right] = temp;
  return track;
};

const selectIdx = (arr, left, right, k, compare) => {
  if (left === right) {
    return left;
  }
  let dest = left + k;
  while (true) {
    let pivotIndex =
      right - left + 1 <= 5
        ? Math.floor(Math.random() * (right - left + 1)) + left
        : medianOfMedians(arr, left, right, compare);
    pivotIndex = partition(arr, left, right, pivotIndex, compare);
    if (pivotIndex === dest) {
      return pivotIndex;
    } else if (pivotIndex < dest) {
      left = pivotIndex + 1;
    } else {
      right = pivotIndex - 1;
    }
  }
};

const medianOfMedians = (arr, left, right, compare) => {
  let numMedians = Math.ceil((right - left) / 5);
  for (let i = 0; i < numMedians; i++) {
    let subLeft = left + i * 5;
    let subRight = subLeft + 4;
    if (subRight > right) {
      subRight = right;
    }
    let medianIdx = selectIdx(arr, subLeft, subRight, Math.floor((subRight - subLeft) / 2), compare);
    let temp = arr[medianIdx];
    arr[medianIdx] = arr[left + i];
    arr[left + i] = temp;
  }
  return selectIdx(arr, left, left + numMedians - 1, Math.floor(numMedians / 2), compare);
};

const defaultCompare = (a, b) => {
  return a < b ? -1 : a > b ? 1 : 0;
};
```
