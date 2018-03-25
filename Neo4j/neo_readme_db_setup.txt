1) Copy all data files from zip folder to <neo_db path>\import\

2)  The following are the queries to load the data into Neo Graph DB
----------------------------User data Upload with Id as key-----------------------------
LOAD CSV WITH HEADERS FROM "file:///Users.CSV" AS row
WITH row LIMIT 20

MERGE (user:User{id:row.Id})
ON CREATE SET user.creation_date=row.CreationDate,
			  user.display_name=row.DisplayName,
			  user.views=row.Views,
			  user.up_votes=row.UpVotes,
			  user.down_votes=row.DownVotes;

----------------------------Question data upload with Id as key--------------------------
LOAD CSV WITH HEADERS FROM "file:///Questions.CSV" AS row
WITH row LIMIT 20

MERGE (ques:Question{id:row.Id})
ON CREATE SET ques.accepted_answer_id=row.AcceptedAnswerId,
			  ques.q_creation_date=row.CreationDate, 
			  ques.view_count=row.ViewCount, 
			  ques.owner_user_id=row.OwnerUserId, 
			  ques.title=row.Title, 
			  ques.tags=row.Tags,
			  ques.owner_display_name=row.OwnerDisplayName;
			  
----------------------------Answers data upload with Id as key-----------------------------
LOAD CSV WITH HEADERS FROM "file:///Answers.CSV" AS row
WITH row LIMIT 20

MERGE (ans:Answer{id:row.Id})
ON CREATE SET ans.parent_id=row.ParentId,
			  ans.a_creation_date=row.CreationDate,  
			  ans.owner_user_id=row.OwnerUserId;

----------------------------Votes data upload with Id as key-----------------------------
LOAD CSV WITH HEADERS FROM "file:///Votes.CSV" AS row
WITH row LIMIT 20

MERGE (vote:Vote{id:row.Id})
ON CREATE SET vote.post_id=row.PostId,
			  vote.vote_type_id=row.VoteTypeId,  
			  vote.v_creation_date=row.CreationDate;

----------------------------Tags data upload with Id as key-----------------------------
LOAD CSV WITH HEADERS FROM "file:///Tags.CSV" AS row

MERGE (tag:Tag{id:row.Id})
ON CREATE SET tag.tag_name=row.TagName,
			  tag.tag_count=row.Count;
			  
-------------------------------INDEXING---------------------------------------------------
Create index on: User(id)

Create index on:Question(id)
Create index on:Question(accepted_answer_id)






---------------------------Creating User->Question relation-------------------------------			  
MATCH (user:User),(ques:Question)
WHERE user.id=ques.owner_user_id
MERGE (ques)-[:ASKED_BY]->(user);

MATCH (user:User),(ques:Question)
WHERE user.id=ques.owner_user_id
MERGE (user)-[:ASKED]->(ques);

MATCH (user:User),(ans:Answer)
 WHERE user.id=ans.owner_user_id
MERGE (ans)-[:ANSWERED_BY]->(user);

MATCH (user:User),(ans:Answer)
 WHERE user.id=ans.owner_user_id
MERGE (user)-[:ANSWERES]->(ans);

MATCH (ques:Question),(ans:Answer)
 WHERE ques.id=ans.parent_id
MERGE (ques)-[:HAS_ANSWER]->(ans);

MATCH (ques:Question),(ans:Answer)
 WHERE ques.id=ans.parent_id
MERGE (ans)-[:ANSWERS_THIS]->(ques);

MATCH (ques:Question),(vote:Vote)
WHERE ques.id=vote.post_id
MERGE (ques)-[:QVOTES]->(vote);

MATCH (ans:Answer),(vote:Vote)
WHERE ans.id=vote.post_id
MERGE (ans)-[:AVOTES]->(vote);

MATCH (ques:Question)
FOREACH (tagName IN split (ques.tags, ";") |
MERGE (tag:Tag {tag_name:tagName})
MERGE (tag)<-[:HAS_TAG]-(ques));

MATCH (ques:Question)
FOREACH (tagName IN split (ques.tags, ";") |
MERGE (tag:Tag {tag_name:tagName})
MERGE (tag)-[:TAGGED_TO]->(ques))
---------------------------------------------------------------------------
---------------------------------1st simple query----------------------------------------------------------------
Match (ques:Question{id:"7"})-[:ASKED_BY]->(n:User),(ans:Answer{parent_id:"7"})-[:ANSWERED_BY]-(u:User)
return distinct n,u
---------------------------------2nd simple query----------------------------------------------------------------
Match p=(t:Tag{tag_name:'data-request'})-[:TAGGED_TO]->(q:Question) with  max(toInteger(q.view_count)) as max_views
match x=(ques:Question)
where toInteger(ques.view_count)=max_views
return toInteger(ques.view_count), ques.title;

---------------------------------1st analytical query----------------------------------------------------------------
Match p=(t:Tag)-[:TAGGED_TO]->(q:Question)-[:HAS_ANSWER]->(a:Answer)
where t.tag_name IN['usa','fcc','api','city']
and exists(q.accepted_answer_id) and q.accepted_answer_id=a.id
return toInteger(a.a_creation_date)-toInteger(q.q_creation_date) as diff,q.title,q.id,t.tag_name order by diff

--------------------------------------2nd analytical query----------------------------------------------------------------
Match (t:Tag)-[:TAGGED_TO]->(q:Question)-[:HAS_ANSWER]->(a:Answer)-[:ANSWERED_BY]->(u:User)
where (toInteger(q.q_creation_date)>= 1357027578) and (toInteger(q.q_creation_date)<= 1388563578) and (toInteger(a.a_creation_date)>= 1357027578) or (toInteger(a.a_creation_date)<= 1388563578)
with distinct u.id as person,(t.tag_name) as topic
return  topic, count(topic) as no_users order by no_users desc

-------------------------------------3rd analytical query----------------------------------------------------------------
match (t:Tag{tag_name:"api"} )-[:TAGGED_TO]->(q:Question),(q:Question)-[:HAS_ANSWER]->(a:Answer)
where exists(q.accepted_answer_id) and  (q.accepted_answer_id= a.id)
//match (a:Answer)-[:ANSWEREDBY]->(u:User)
with  (a.owner_user_id) as User, q.title as ques
return User,count(User) as total, collect(distinct ques) [0..] as Allques
order by total desc limit 1

-------------------------------------4th analytical query----------------------------------------------------------------
MATCH p=(u:User)-[r:ANSWERES]->(a:Answer)-[:ANSWERS_THIS]->(q:Question)-[:HAS_TAG]->(t:Tag)
where u.id="1511" and u.id=a.owner_user_id and a.parent_id=q.id and a.id=q.accepted_answer_id
with  t.tag_name as tag, count(t.tag_name) as tag_count 
where tag_count >= 5
with tag as top_tags
match o=(t1:Tag)-[:TAGGED_TO]->(q1:Question)
where t1.tag_name in top_tags and not exists(q1.accepted_answer_id)
with distinct q1.title as quest, q1.q_creation_date as created_date
return quest, created_date order by created_date desc limit 5;

-------------------------------------6th analytical query----------------------------------------------------------------
match(u:User)-[:ANSWERES]->(a:Answer)-[:ANSWERS_THIS]->(q:Question)-[:HAS_ANSWER]->(a1:Answer)-[:ANSWERED_BY]->(u1:User)
where (a.owner_user_id="1511")
with collect({uid:u1.id}) as rows
match(u2:User)-[:ASKED]->(q1:Question)-[:HAS_ANSWER]->(a2:Answer)-[:ANSWERED_BY]->(u3:User)
where (q1.owner_user_id="1511")
with rows + collect({uid:u3.id})  as all_rows
unwind all_rows as row
return row.uid, count(row.uid) as couser_count order by couser_count desc skip 1 limit 5;






