### Tasks

## #1

Requests to Acme's API are timing out after 15 seconds. After some investigation in production, they've identified that the `User#count_hits` method's response time is around 500ms and is called 50 times per second (on average).

### Proposed Solution:
The easiest solution here is to use counter cache which is a built in feature for Rails. This will save the DB calls which are currently being
made. For future; We could use Redis if needed. But at the moment it does not matter as current user is already present, with Redis we can fail even before loading the user.

The implementation of counter cache is as simple as defining it in the Hit model and adding the column into the user table
```ruby
class Hit < ApplicationRecord
  belongs_to :user, counter_cache: true
end
```
The migration would be as follows

```shell
rails generate migration AddHitsCountToUsers hits_count:bigint
rails db:migrate
```
After this we would not need to use `User#count_hits`, and we could reference the `hits_count` column directly and avoid the DB call as below.

* Note: We do need to reset the counter every 1 month as described below in the doc.

```ruby
class ApplicationController < ActionController::API
  before_filter :user_quota

    def user_quota
     render json: { error: 'over quota' } if current_user.hits_count >= 10000 # TODO: Change 10000 to be a constant defined in the business layer as applicable
    end
end
```
As mentioned before, We would need to reset the counter every end of month to give the quota back to user. Rather then running a cron job
every day to know which user to reset the quota for (Since we are moving to timezone based reset which is the problem#2). We schedule jobs to run
at end of the month in users timezone after every save; (If timezone field is changed)

Called after save and not create to cater the case in case user changes his timezone within a running month.

The job would look like this:

```ruby
module Hits
  class ResetUsersHitsCountJob < ApplicationJob
    queue_as :default
    
    def perform(user_id)
      user = User.find(user_id)
      user&.reset_hits_count
    end
    
  end
end
```

We can schedule this job to run at the beginning of every month for each user and this would reset the counter_cache to 0 for that user by calling the method defined below
```ruby
class UserQuotaLimit < ApplicationRecord
  # Moved to interactors as mentioned in the end.
  def reset_hits_count
    update_column(:hits_count, 0)
  end
  
  # Setting the time to run at users 00:00:00 at start of the month
  def schedule_next_reset_job
    job = Hits::ResetUsersHitsCountJob.set(
      wait: DateTime.now.at_beginning_of_month.next_month.in_time_zone(timezone)
    ).perform_later(id)
    
    update_column(:reset_hits_job_id, job.id)
  end
  
end
```

```ruby
class User < ApplicationRecord
  include UserQuotaLimit
  
  # This can be removed if Interactor based implementation is done as mentioned at the end.
  after_save :schedule_next_reset_job, if: -> { hits_count.to_i.zero? } # Run only when we either reset back to 0 or on initial create
  validates :timezone, presence: true # Validation added to ensure we have timezone. We can default in DB as per business flow
end
```

## #2

Users in Australia are complaining they still get some “over quota” errors from the API after their quota resets at the beginning of the month and after a few hours it resolves itself. A user provided the following access logs:
```shell
timestamp                 | url                        | response
2022-10-31T23:58:01+10:00 | https://acme.corp/api/g6az | { error: 'over quota' }
2022-10-31T23:59:17+10:00 | https://acme.corp/api/g6az | { error: 'over quota' }
2022-11-01T01:12:54+10:00 | https://acme.corp/api/g6az | { error: 'over quota' }
2022-11-01T01:43:11+10:00 | https://acme.corp/api/g6az | { error: 'over quota' }
2022-11-01T16:05:20+10:00 | https://acme.corp/api/g6az | { data: [{ ... }] }
2022-11-01T16:43:39+10:00 | https://acme.corp/api/g6az | { data: [{ ... }] }
```

### Problem:
Australia is ahead of UTC / EST and etc. So their month starts before others; And we use Time directly without Timezone. So its not updated until a few hours until
00:00:00 is reached in UTC

### Proposed Solution:
We already resolved this issue which is happening before Australia is ahead of UTC and server time is different (UTC most likely); To
handle this i introduced the timezone field which we use to schedule jobs in users timezone at 00:00:00 on 1st of every month rather than
checking Time directly.

The migration is as follow; We can add a default as per business flow.
```shell
rails generate migration AddTimezoneToUsers timezone:string
rails db:migrate
```

After this; If you look at the User model. You can see we call a callback to schedule jobs in users timezone.

#### * Note (Needs Discussion)
We could and should store the Job ID; The reason being if the user changes his/her timezone within a month cycle; We might want to remove and enqueue new one
but that would mean he started on a different quota and finish on another; Would need discussion with stakeholders on how to handle this case.


For the migration to add the new field, we would need to run:
```shell
rails generate migration AddResetHitsJobIdToUsers reset_hits_job_id:integer
rails db:migrate
```

## #3

Acme identified that some users have been able to make API requests over the monthly limit.

### Problem:
Again, The issue would arise for companies which are behind UTC. For them the quota is refreshed due to the fact we do not check UTC.
This is already resolved in the above implementation. There might be a small issue for concurrent requests; Which is very minor; But can be handled by locking rows at the time Hits are being saved.

### Solution:
Already resolved in the implementation.


## #4
Acme's development team has reported working with the code base is difficult due to accumulated technical debt and bad coding practices. They've asked the community to help them refactor the code so it's clean, readable, maintainable, and well-tested.


### Solution
Do not have full insight into the code base, But the minimal stuff we have. I would recommend using the following. Right now you can see I
created concerns in the models to keep the reset/quota logic confined into a concern module rather then keeping fat model.
The change I would make there is to move stuff into Interactors (https://github.com/collectiveidea/interactor). I have been using for a
while and it's a really clean way of keeping dry classes in your repo. Basically rather than doing after_save and other stuff. I would chain
interactors via their organizer concept. Which is really clean and easy to manage in long term. Example is below. And of-course we need to add
rspec coverage for unit and acceptance to ensure all this time zone/ job ID stuff is working as expected.

Secondly, I would recommend not using SQL to store Hits data; It is going to expand vertically. So NOSQL might be a better approach.
So where ever which is not apparent from the current code base Hits are being saved; I think NoSQL might be a better fit. Redis is also
a good option but then we have persistency issue(s) with that.

I did implement a cloud based solution for this for a company to save Hits data as analytics; Happy to discuss a cloud based approach
if needed.

The 10000 limit should be moved to the user level to make it more dynamic; So if in future we want to give different limits
to different users based on plans or something; We can do that easily. At the very least; Move it to ENV variable. So we can
modify that for easy testing of different scenarios.

We need proper logging; e.g with Hits save IP and other information for book keeping purposes; If client says I did not use that much :)
And error reporting for failures into something like Rollbar or similar.

```ruby
module ApiUserQuota
  class Reset
    include Interactor::Organizer
    delegate :user, to: :context
    
    # Its a single purpose and updated the current context and passes auto to next one; If 1 fails. Next would not run. Pretty Neat ! 
    organize ApiUserQuota::ResetCounter, # Reset the DB hits_count column
             ApiUserQuota::ScheduleNextJob, # Responsible to scheduling the next job
             ApiUserQuota::SaveNextJob # Responsible to save the job ID into users
  end
end

```