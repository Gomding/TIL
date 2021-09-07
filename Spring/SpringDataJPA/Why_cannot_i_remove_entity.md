# Entity 객체 삭제 왜 원하는대로 동작을 안할까?

# 목차
* [개요](#개요)
* [테스트 모델](#테스트모델)
* [CascadeType.ALL](#CascadeType.ALL)
* [결론](#결론)

# 개요
팀 프로젝트에서 댓글 삭제 기능을 구현하다가 벽을 만났습니다. 댓글을 삭제하면 관계된 데이터(해당 댓글에 달린 대댓글, 댓글에 추가된 좋아요)를 함께 삭제하고 싶었습니다.  
때문에 CascadeType.REMOVE 와 orphanRemoval=true 옵션을 사용했습니다.  
하지만 원하는 것처럼 댓글 삭제시 관계된 객체들이 함께 삭제되기는 커녕 댓글도 삭제가 되지 않았습니다.  
왜 그런걸까요? 먼저 테스트 모델을 설명하겠습니다.

# 테스트 모델
유저(**User**)가 있고 글(**Feed**), 댓글(**Comment**), 댓글 좋아요(**CommentLike**)가 존재합니다.  
댓글에 대한 대댓글은 자기 참조 테이블처럼 자기 자신을 참조하는 것으로 **List< Comment > reply** 필드로 가지고 있습니다.  
![Untitled Diagram](https://user-images.githubusercontent.com/57378410/132387072-428f287f-90aa-4960-bded-f9417508bf16.png)


```java
@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private Integer age;

    @OneToMany(mappedBy = "user")
    private List<Feed> feeds = new ArrayList<>();

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<CommentLike> commentLikes = new ArrayList<>();

    // Constructor

    public void addFeed(Feed feed) {
        this.feeds.add(feed);
    }

    public void addComment(Comment comment) {
        this.comments.add(comment);
    }

    public void deleteComment(Comment comment) {
        this.comments.remove(comment);
    }

    public void addCommentLike(CommentLike commentLike) {
        this.commentLikes.add(commentLike);
    }

    // getter
}
```
```java
@Entity
public class Feed {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    private String content;

    @ManyToOne
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @OneToMany(mappedBy = "feed", cascade = CascadeType.REMOVE)
    private List<Comment> comments;

    // Constructor

    public void writtenBy(User user) {
        this.user = user;
    }

    public void addComment(Comment comment) {
        this.comments.add(comment);
        comment.setFeed(this);
    }

    // getter
}
```

```java
@Entity
public class Comment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String content;
    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @ManyToOne
    @JoinColumn(name = "feed_id", nullable = false)
    private Feed feed;

    @ManyToOne
    @JoinColumn(name = "parent_id")
    private Comment parent;

    @OneToMany(mappedBy = "parent", cascade = CascadeType.REMOVE, orphanRemoval = true)
    private List<Comment> replies;

    @OneToMany(mappedBy = "comment", cascade = CascadeType.REMOVE, orphanRemoval = true)
    private List<CommentLike> commentLikes;

    // Constructor

    public void writtenBy(User user) {
        this.user = user;
        user.addComment(this);
    }

    public void addReply(Comment reply) {
        replies.add(reply);
        reply.setParent(this);
    }

    public void addCommentLike(CommentLike commentLike) {
        this.commentLikes.add(commentLike);
    }

    public void deleteAllReply() {
        this.replies.forEach(reply -> reply.parent = null);
    }

    public void setFeed(Feed feed) {
        this.feed = feed;
    }

    private void setParent(Comment comment) {
        this.parent = comment;
    }

    // getter
}
```
```java
@Entity
public class CommentLike {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne
    @JoinColumn(name = "user_id")
    private User user;

    @ManyToOne
    @JoinColumn(name = "comment_id")
    private Comment comment;

    // Constructor

    public static CommentLike of(User user, Comment comment) {
        CommentLike commentLike = new CommentLike(user, comment);
        user.addCommentLike(commentLike);
        comment.addCommentLike(commentLike);
        return commentLike;
    }

    // getter
}
```

# CascadeType.ALL
```java
    @Test
    void remove() {
        // given
        User savedUser1 = userService.save(user1);
        User savedUser2 = userService.save(user2);
        Feed savedFeed1 = feedService.save(user1, feed1);

        Comment comment = new Comment("댓글 내용");
        comment.writtenBy(savedUser2);
        savedFeed1.addComment(comment);
        Comment savedComment = commentService.save(comment);

        Comment reply = new Comment("대댓글 내용");
        reply.writtenBy(savedUser1);
        savedFeed1.addComment(reply);
        savedComment.addReply(reply);
        commentService.save(reply);

        entityManager.flush();

        commentRepository.delete(savedComment);

        entityManager.flush();
    }
```
위와 같이 테스트를 진행했을때 ```commentRepository.delete(savedComment);```를 했음에도 댓글이 삭제되지 않음을 확인했습니다. 댓글을 삭제하는 delete 쿼리 또한 날아가지 않았습니다.  
문제는 User 객체에 Comment와 OneToMany 관계에 있는 comments에 설정한 CascadeType.ALL 이었습니다.  
```java
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();
```
CascadeType.ALL에는 CascadeType.PERSIST가 포함되어 있습니다.  
때문에 Comment 객체를 삭제하려고 객체의 상태를 Deleted 로 바꿔줘도 User 엔티티에 설정된 CascadeType.PERSIST가 동작하여 Deleted 상태를 Persist로 바꿔주는 것입니다. 
* commentRepository.delete(savedComment); -> savedComment 객체의 상태를 Deleted로 설정합니다.
* flush 될 때 User 객체의 comments 필드에 CascadeType.ALL 설정에 의해 savedComment 객체는 Deleted상태에서 Persist상태로 변경됩니다.(정확히는 CascadeType.PERSIST 때문)  
* savedComment 객체의 상태가 Persist 이기 때문에 삭제가 진행되지 않습니다.


# 결론
이런 문제를 방지하기 위해서는 CascadeType.ALL 설정을 함부로 사용하면 안된다는 것입니다.
CascadeType 설정에 대한 지식이 부족해서 발생한 문제였습니다.  
실제로 저는 CascadeType 설정에 대한 문제라고 생각하지 못하고 객체의 관계를 하나하나 끊어주고 삭제를 진행했습니다.
이렇게해도 삭제는 잘 되지만 올바른 사용법이 아니라고 생각합니다.  
지식이 부족한것을 JPA라는 기술에다가 불편하다고 했던 자신이 부끄럽네요 ..
현재는 CascadeType.REMOVE 와 orphanRemoval=true 옵션을 사용해서 삭제를 진행하고 있습니다. 
