---
title: "高性能React数据容器WatermelonDB"
date: 2018-11-07T00:00:00+08:00
---

## 简介

WatermelonDB 是 React Native 和 React Web 一种新的数据状态容器，虽然称作 DB 但意义和传统 DB 还是有些区别的，它的作用更像 Redux，同 Redux 类似，WatermelonDB 也是单一数据源原则，并且数据流单向的形式，UI 会根据记录的变化而相应的改变。
WatermelonDB 比 Redux 的优势在于，对于数据记录较大的 React 应用，能提供更好的性能表现。同时 WatermelonDB 也能比 Redux 更友好的管理数据流。

## 示例

首先，定义 Models

```
class Post extends Model {
  @field('name') name
  @field('body') body
  @children('comments') comments
}

class Comment extends Model {
  @field('body') body
  @field('author') author
}
```

然后 connect 数据到 components（和 Redux connect 比较像）

```
const Comment = ({ comment }) => (
  <View style={styles.commentBox}>
    <Text>{comment.body} — by {comment.author}</Text>
  </View>
)

// This is how you make your app reactive! ✨
const enhance = withObservables(['comment'], ({ comment }) => ({
  comment: comment.observe()
}))
const EnhancedComment = enhance(Comment)
```

render

```
const Post = ({ post, comments }) => (
  <View>
    <Text>{post.name}</Text>
    <Text>Comments:</Text>
    {comments.map(comment =>
      <Comment key={comment.id} comment={comment} />
    )}
  </View>
)

const enhance = withObservables(['post'], ({ post }) => ({
  post: post.observe(),
  comments: post.comments.observe()
}))
```

现在添加、更新、删除操作一条数据记录，绑定的 component 就会自动重新 render 了。
对于数据流大、应用复杂又比较追求性能的 React 应用，可以尝试一下 WatermelonDB 了。

安装文档：https://github.com/Nozbe/WatermelonDB/blob/master/docs/Installation.md
