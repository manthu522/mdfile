
# Commenting guidelines:

## 1. Running previous scripts
###	1a. Run the liquibacse script for latest db
- navigate to the clamour-db folder
- Run the script "liquibaseUpdate.sh hostname db-schema-name username password update"

###  1a. Script for GeoChannels(group_channels in database)
- Purpose: To create Geo Channels for all the groups present in our database
- Route: /comments/scriptForGeoChannels
- Controller: scriptForGeoChannels function in Comments controller(controllers/comments)
  - Fetching all the groups which are not associated with any group_channel from models/group and calling  'channelToGroups' function
  - Iterating the groups and preparing group_channel_name(id+"_geo_channel") and inserting into `channels`(name) table and inserting into `group_channel`(channels_id, groups_id and modified) table

###  1b. Script for ChannelGroups(channel-groups in database)
- Purpose: Create channelGroups for all users and subscribe to respective group channel
- Route: /comments/scriptForChannelGroups
- Controller: scriptForChannelGroups function in Comments controller(controllers/comments)
  - Fetching all the users which from users table for whom deleted flag is not set
  - Iterating the users and creating  channel_group_name(userid+"_channel_group") and calling createchannelgroup function from comments controller by passing usersId, channel_group_name and groups_id to it, here we are getting the id, name(group channel name) from `channels` table based on the groups_id , inserting into `channelgroups_has_channels` table and `users_has_channelgroups` table with respective columns.
- Pubnub: Calling channelGroupAddChannel function from app/helpers/pubnub file, and pass channelName(Geo channel==group channel) and chanelGroup Name to it, and in the function we need to add channel to channel group, below is the code snippet how to add a channel to channel group
```javascript
pubnub.channel_group_add_channel({
  channel   : channelName,
  channel_group: channelGroupName
}, function(results){
  callback(results);
});
```
###  1c. Script for PostChannels
- Purpose: Fetch all posts which are not associated with any posts channel(previous posts)
- Route: /comments/scriptForPostChannels
- Controller: scriptForPostChannels function in Comments controller(controllers/comments)
  - Fetching all the previous posts from posts table from db, and preparing post_chaannel_name(posts_id+"_post_channel") and calling createpostchannel function from comments controler by passing post_channel_name and posts_id to it and inserting into `channels`(name==post_channel_name) table and inserting into `post_channel`(channels_id, groups_id and modified) table and also inserting into `channelgroups_has_channels` table
    - Pubnub: Calling channelGroupAddChannel function from app/helpers/pubnub file, and passing channelName(Post channel name) and chanelGroup Name to it, and in the function we need to add channel(post_channel) to channel_group, below is the code snippet how to add a channel to channel group
```javascript
pubnub.channel_group_add_channel({
  channel   : channelName,
  channel_group: channelGroupName
}, function(results){
  callback(results);
});
```
### 1d. Running the contineous script
- Purpose: To fetch all the previous comments from fb and stores into post_has_caomments table, fb_comments table and also publishing to appropriate channels(post channels)  by using PUBNUB publish api call , here is code snippet
```javascript
pubnub.publish({
  channel : postId+'_'+'post_channel',
  message : { 'message' :message, 'image_url' : image},
  callback : function(e) {
    logger.info("SUCCESS from publishing the channel", e[1], e[0], e[2] );
    callback("SUCCESS");
  },
  error : function(e) {
    logger.info( "FAILED! RETRY PUBLISH!", e ); 
    callback("ERROR");
  }
});
```
- We need to run the continuous script form the project root directory by using following command 
NODE_ENV=dev NODE_PAth=. PORT=80 node scripts/fbPubNubSync.js

##### *Note : you can use REST-Client to hit the routes, and these all api calls of type GET and don't need any request params to be sent from clientside

## 2. Current commenting system flow

### while creating a new group in our system(fb group or clamour group)
- Route : /groups
- Controller : group 
  - in this controller we are caling 'create' function(if the group is clamour group), in  which  we are caling  'create' function from groups model to create a new group in our db and calling createClamourGeoChannel from group model to create a new group_channel( inserting into `channels`(name) table and inserting into `group_channel`(channels_id, groups_id and modified) table)
  - in this controller we are caling 'createWithFb' function(if the group is fb group), in  which  we are caling createGroup function from fb file(app/lib/fb) to create a new group in fb using fb api, calling 'create' function from groups model to create a new group in our db and calling createClamourGeoChannel from group model to create a new group_channel( inserting into `channels`(name) table and inserting into `group_channel`(channels_id, groups_id and modified) table)

### while user joining into a group
- Under progress

### while user posting a new post
- Route : /groups/:groupid/posts
- Controller : post
  - in this we are calling 'create' function, in which we are calling 'create' function from post model to store a new post in our db, if the group is fb groups also calling 'postToGroup'(from app/lib/facebook_api) function to store the post in fb usng fb api
  - if it is fb group or non fb group, we are calling 'createpostchannel' function by passing post_channel_name and posts_id to it and inserting into`channels` (name==post_channel_name) table and inserting into `post_channel`(channels_id, groups_id and modified) table and also inserting into `channelgroups_has_channels` table
    - Pubnub: Calling channelGroupAddChannel function from app/helpers/pubnub file, and passing channelName(Post channel name) and chanelGroup Name to it, and in the function we need to add channel(post_channel) to channel_group, below is the code snippet how to add a channel to channel group
```javascript
pubnub.channel_group_add_channel({
  channel   : channelName,
  channel_group: channelGroupName
}, function(results){
  callback(results);
});
```

### while user comments on post
- Route : /comments/postcomment
- Controller : comments
  - in this we are calling 'addcomment' function from comments model, to store the new comments in our db into post_has_caomments table, fb_comments table.
  - if the group is fb group, we are calling 'commentAndImageOnPost' function from the fb.js(app/lib/fb) file, to store the comments on fb usinf fb api.
  - if the group is fb groip ar non fb group, we are calling 'publishByPostId' function from pubnub helper file(app/helpers/pubnub), to publish the message to the post_chennel. here is code snippet for publishing a message to pubnub

```javascript
pubnub.publish({
  channel : postId+'_'+'post_channel',
  message : { 'message' :message, 'image_url' : image},
  callback : function(e) {
    logger.info("SUCCESS from publishing the channel", e[1], e[0], e[2] );
    callback("SUCCESS");
  },
  error : function(e) {
    logger.info( "FAILED! RETRY PUBLISH!", e ); 
    callback("ERROR");
  }
});
```

### getting the comments on posts from our db
- Route : /comments/commentsOnPost/:postid
- req params : post_id
- Controller : we are calling 'commentsOnPost' function for fetching all the comments on the particular post.
