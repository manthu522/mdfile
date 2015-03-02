## Pub Nub - Clamour Implementation

Core implementation of **Pub Nub** in **Clamour** is for receiving notifications for events across Clamour. Pub Nub is implemented by keeping `70%` notification system use cases, `25%` messaging use cases and `5%` commenting use cases. So to understand the flow of implementation it's important to keep in mind we are publishing - subscribing - receiving notification messages.

#### Pub Nub - Commenting System

In commenting system Pub Nub covers the concept of mostly notification system and hence send messages for new event based on user subscription. 
**Ex.: If user Jake comment on a Post Jackass, created by User Alejandro, then here are the steps for it (Fig. 1):**

![Fig 1.](https://dl.dropboxusercontent.com/s/3h5eirvx5hkgazq/fig%201.png)

 * **Alejandro** creates Post `Jackass` -> User **Alejandro** subscribed for all future events related notifications associated with this Post
 * **Jake** comment on Post `Jackass` -> User **Alejandro** subscribed for all future events related notifications for Post `Jackass`
 * **Alejandro** receives a notification with comment published by User **Jake** as **Alejandro** was subscribed to Post `Jackass`.

#### Analogy of Pub Nub W.R.T. Clamour

Pub Nub (represented in code as)                         | Clamour
---------------------------------------------------------| ----------------
Channels (`groupId`_geo_channel / `postId`_post_channel) | Groups and Posts
Channel Groups (`userId`_channel_group)                  | Users

**Limitations and Dependencies from Pub Nub**

1. There can be maximum `2000` **channels** in a **channel group**.
2. There can be any number of **channel groups** associated with an account.
3. There must be at least one **channel** subscribed to a **channel group** at the time of it's creation.
4. A channel can be created individual of any channel group and hence can be subscribed later to any number of channel groups (don't confuse, this is a **Pubnub channel group**) at a time.

To overcome the first two limitations we have followed the approach of representing Users as channel groups, groups and posts as channels. Where users can subscribe to `2000` channels (including posts and groups) in a single channel group and a user can have multiple channel groups.

Considering the third limitation (more accurately dependancy) we have not created channel groups right away after user registration, instead by following the forth point we have first created channels for all posts and all groups. Then later when users joins a group, at that time we are creating a channel group for user as we have a channel to subscribe too. **Guess which channel ???** The channel of group which is joined by user.

**Let's re-consider the User Jake and Alejandro example again following the above discussed points as follows:**

###### Users
###### ------
1. Alejandro  ----PUBNUB----  1_channel_group
2. Jake  ----PUBNUB----  2_channel_group

###### Groups
###### -------
1. Group A  ----PUBNUB----  1_geo_channel
2. Group B  ----PUBNUB----  2_geo_channel
3. Group C  ----PUBNUB----  3_geo_channel
4. Group D  ----PUBNUB----  4_geo_channel
5. Clamour Podcast  ----PUBNUB----  5_geo_channel

###### Posts
###### ------
1. Jackass  ----PUBNUB----  1_post_channel
2. P (Post), P, ... P (n), P (n+1) :wink:

**Fig 2 : 10,000 Feet Logical Approach**

![Fig 2.](https://dl.dropboxusercontent.com/s/vasude3b1fjv96a/fig%202.png)

**Fig 3 : Approach with Proper Terminologies and detailed with code snapshots in description**

![Fig 3.](https://dl.dropboxusercontent.com/s/vosh5vpj1uo1eem/fig3.png)

As in fig 3. first Alejandor joins group Clamour Podcast which has channel `5_geo_channel`, hence a new channel group is created for Alejandro `1_channel_group` based on user ID and subscribed channel `5_geo_channel` using code snippet -
  
  ```javascript
  // Subscribe channel to chnnel group
  pubnub.channel_group_add_channel({
    channel   : 5_geo_channel,
    channel_group: 1_channel_group
  }, function(results){
      callback(results);
  });
  ```

Similarly **Jake** is also subscribed to the same group channel (`5_geo_channel`) in his own channel group - `2_channel_group`, which illustrates a channel can be subcribed to more than one channel groups.

Now as soon as **Alejandro** created a new post in group **Clamour Podcast**, a new post channel (`1_post_channel`) is created and it's subscribed immediately to Alejandro's channel group as -

  ```javascript
  // Subscribe new Post channel to channel group
  pubnub.channel_group_add_channel({
    channel   : 1_post_channel,
    channel_group: 1_channel_group
  }, function(results){
      callback(results);
  });
  ```

**Here if you have noticed we haven't called for seperate pubnub service for creating a post channel first. For performance optimization it's best to pass the channel name in the subscription call itself and pubnub will create the channel itself (if not exist) and then subscribe to specified channel group.**

When **Jake** come to **like** the post he is subscribed in the similar fashion as above to his channel group - 

  ```javascript
  // Subscribe new Post channel to channel group
  pubnub.channel_group_add_channel({
    channel   : 1_post_channel,
    channel_group: 2_channel_group
  }, function(results){
      callback(results);
  });
  ```

As soon as **Jake** likes the post a message published to channel `1_post_channel` for this and hence all the channel_groups those hold the subscription for channel `1_post_channel` will receive the notification. Here is the code snapshot for publishing message - 

  ```javascript
  // Publish by Post ID
  pubnub.publish({
    channel : postId+'_'+'post_channel',
    message : { 'message' :message, 'image_url' : imageUrl}
  }, function(results){
    callback(results);
  });
  

  // Publish by Group ID
  pubnub.publish({
    channel : groupId+'_'+'geo_channel',
    message : { 'message' :message, 'image_url' : imageUrl}
  }, function(results){
    callback(results);
  });

  // Publish by Channel Name
  pubnub.publish({
    channel : channelName',
    message : { 'message' :message, 'image_url' : imageUrl}
  }, function(results){
    callback(results);
  });
  ```


#### DB Tables Description

###### Pub Nub
###### ---------
* channels
* group_channel
* post_channel
* channelgroups_has_channels
* users_has_channelgroups

###### Commenting System
###### ------------------
* cat_tags
* comment_access_rights
* comments_has_flags
* comments_has_users_tags
* fb_comments
* posts_has_comments
* posts_has_comments_has_cat_tags


#### Process to Setup the new commenting system with Pub Nub


1. Make sure you have latest DB from liquibase 

2. Script for GeoChannels (group_channels in database)
 * Purpose: To create Geo Channels for all the groups present in our database
 * Route: /comments/scriptForGeoChannels
 * Controller: scriptForGeoChannels function in Comments controller(controllers/comments)
   * Fetching all the groups which are not associated with any group_channel from models/group and calling  'channelToGroups' function
   * Iterating the groups and preparing group_channel_name(id+"_geo_channel") and inserting into `channels`(name) table and inserting into `group_channel`(channels_id, groups_id and modified) table

3. Script for ChannelGroups(channel-groups in database)
 * Purpose: Create channelGroups for all users and subscribe to respective group channel
 * Route: /comments/scriptForChannelGroups
 * Controller: scriptForChannelGroups function in Comments controller(controllers/comments)
   * Fetching all the users which from users table for whom deleted flag is not set
   * Iterating the users and creating  channel_group_name(userid+"_channel_group") and calling createchannelgroup function from comments controller by passing usersId, channel_group_name and groups_id to it, here we are getting the id, name(group channel name) from `channels` table based on the groups_id , inserting into `channelgroups_has_channels` table and `users_has_channelgroups` table with respective columns.
   * Pubnub: Calling channelGroupAddChannel function from app/helpers/pubnub file, and pass channelName(Geo channel==group channel) and chanelGroup Name to it, and in the function we need to add channel to channel group, below is the code snippet how to add a channel to channel group

  ```javascript
  pubnub.channel_group_add_channel({
    channel   : channelName,
    channel_group: channelGroupName
  }, function(results){
    callback(results);
  });
  ```
4. Script for PostChannels
 * Purpose: Fetch all posts which are not associated with any posts channel(previous posts)
 * Route: /comments/scriptForPostChannels
 * Controller: scriptForPostChannels function in Comments controller(controllers/comments)
   * Fetching all the previous posts from posts table from db, and preparing post_chaannel_name(posts_id+"_post_channel") and calling createpostchannel function from comments controler by passing post_channel_name and posts_id to it and inserting into `channels`(name==post_channel_name) table and inserting into `post_channel`(channels_id, groups_id and modified) table and also inserting into `channelgroups_has_channels` table
    - Pubnub: Calling channelGroupAddChannel function from app/helpers/pubnub file, and passing channelName(Post channel name) and chanelGroup Name to it, and in the function we need to add channel(post_channel) to channel_group, below is the code snippet how to add a channel to channel group

  ```javascript
  pubnub.channel_group_add_channel({
    channel   : channelName,
    channel_group: channelGroupName
  }, function(results){
    callback(results);
  });
  ```

5. Running the contineous script
 * Purpose: To fetch all the previous comments from fb and stores into post_has_caomments table, fb_comments table and also publishing to appropriate channels(post channels)  by using PUBNUB publish api call , here is code snippet

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

 * We need to run the continuous script form the project root directory by using following command 

    NODE_ENV=dev NODE_PAth=. PORT=80 node scripts/fbPubNubSync.js

**Note : you can use REST-Client to hit the routes, and these all api calls of type GET and don't need any request params to be sent from clientside**

#### Current commenting system flow

**While creating a new group in our system(fb group or clamour group)**
 * Route : /groups
 * Controller : group 
   * in this controller we are caling 'create' function(if the group is clamour group), in  which  we are caling  'create' function from groups model to create a new group in our db and calling createClamourGeoChannel from group model to create a new group_channel( inserting into `channels`(name) table and inserting into `group_channel`(channels_id, groups_id and modified) table)
   * in this controller we are caling 'createWithFb' function(if the group is fb group), in  which  we are caling createGroup function from fb file(app/lib/fb) to create a new group in fb using fb api, calling 'create' function from groups model to create a new group in our db and calling createClamourGeoChannel from group model to create a new group_channel( inserting into `channels`(name) table and inserting into `group_channel`(channels_id, groups_id and modified) table)

**While user joining into a group**
 * Under progress

**While user posting a new post**
 * Route : /groups/:groupid/posts
 * Controller : post
   * in this we are calling 'create' function, in which we are calling 'create' function from post model to store a new post in our db, if the group is fb groups also calling 'postToGroup'(from app/lib/facebook_api) function to store the post in fb usng fb api
   * if it is fb group or non fb group, we are calling 'createpostchannel' function by passing post_channel_name and posts_id to it and inserting into`channels` (name==post_channel_name) table and inserting into `post_channel`(channels_id, groups_id and modified) table and also inserting into `channelgroups_has_channels` table
     * Pubnub: Calling channelGroupAddChannel function from app/helpers/pubnub file, and passing channelName(Post channel name) and chanelGroup Name to it, and in the function we need to add channel(post_channel) to channel_group, below is the code snippet how to add a channel to channel group

  ```javascript
  pubnub.channel_group_add_channel({
    channel   : channelName,
    channel_group: channelGroupName
  }, function(results){
    callback(results);
  });
  ```

**While user comments on post**
 * Route : /comments/postcomment
 * Controller : comments
   * in this we are calling 'addcomment' function from comments model, to store the new comments in our db into post_has_caomments table, fb_comments table.
   * if the group is fb group, we are calling 'commentAndImageOnPost' function from the fb.js(app/lib/fb) file, to store the comments on fb usinf fb api.
   * if the group is fb groip ar non fb group, we are calling 'publishByPostId' function from pubnub helper file(app/helpers/pubnub), to publish the message to the post_chennel. here is code snippet for publishing a message to pubnub

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

**Getting the comments on posts from our db**
 * Route : /comments/commentsOnPost/:postid
 * req params : post_id
 * Controller : we are calling 'commentsOnPost' function for fetching all the comments on the particular post.

## Authored By ##
* **Dipesh Bhardwaj** - https://github.com/Dev-Dipesh
* **Hemanth Hemu** - https://github.com/manthu522

**Date** - **02 March 2015**
