# Chronofage

Chronofage is a Cron (or whatever scheduler you want) and ActiveRecord based ActiveJob backend. It is
well suited for long-running tasks (heavy video processing, render, etc). Every job
is run in its own process and you can spread the work on multiple hosts.

## Installation

Copy the migration file and run it to create the `chronofage_jobs` table.

```
rake chronofage_engine:install:migrations db:migrate
```

The `chronofage_jobs` table hold informations about all the jobs ready to be executed, failed or done.

```
create_table "chronofage_jobs", force: :cascade do |t|
  t.string   "job_class"
  t.string   "job_id"
  t.string   "queue_name"
  t.text     "arguments"
  t.integer  "priority" .      # follows the `nice` convention (lower value means higher priority)
  t.datetime "started_at"      # set when a job is started
  t.datetime "completed_at"    # set when a job is completed successfuly
  t.datetime "failed_at"       # set when a job failed
  t.datetime "created_at",   null: false
  t.datetime "updated_at",   null: false
end
```

## Usage

You can selectively override the ActiveJob backend for your very long-runny task.

```
class BuildCompilationJob < ApplicationJob
  self.queue_adapter = :chronofage
  queue_as :heavy

  def perform(compilation_settings)
    # long-running stuff
  end
end
```

Then jobs can be run with a simple Rake task taking a queue name and a concurrency argument.

```
rake chronofage_engine:jobs:execute[heavy,2]
```

The task can be run from Cron or any task scheduler you like (the Windows task scheduler, the Heroku scheduler plugin, etc).
The concurrency argument is host-based, so a Cron config like the following one, spread on 3 hosts, will execute a maximum
of 6 jobs, 2 for each host, and check for new jobs every 5 minutes.

```
*/5 * * * * cd /var/www/my_app && rake chronofage_engine:jobs:execute[heavy,2]
```

You can also make Chronofage poll the database for new jobs to execute, this solution is way more reactive.

```
rake chronofage_engine:jobs:poll[heavy,2]
```
