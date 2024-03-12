# Hotwire Todo App

This demo application shows how to use Hotwire & Stimulus to build a basic todo list application in Ruby on Rails.

## Versions

Ruby: v3.2.0
Rails: v.7.1.3.2.

## System dependencies

Ensure that you have [redis](https://redis.io/docs/install/install-redis/) installed.

## Configuration

You'll need to create a `config/master.key` file to work with this application code locally.

## Database creation && initialization

```sh
rails db:create db:migrate
```

## Services

Ensure that Redis is running. On mac with homebrew, try running `brew list redis` to make sure the service is running. You should see something like this:

```sh
/opt/homebrew/Cellar/redis/7.2.4/.bottle/etc/ (2 files)
/opt/homebrew/Cellar/redis/7.2.4/bin/redis-benchmark
/opt/homebrew/Cellar/redis/7.2.4/bin/redis-check-aof
/opt/homebrew/Cellar/redis/7.2.4/bin/redis-check-rdb
/opt/homebrew/Cellar/redis/7.2.4/bin/redis-cli
/opt/homebrew/Cellar/redis/7.2.4/bin/redis-sentinel
/opt/homebrew/Cellar/redis/7.2.4/bin/redis-server
/opt/homebrew/Cellar/redis/7.2.4/homebrew.mxcl.redis.plist
/opt/homebrew/Cellar/redis/7.2.4/homebrew.redis.service
```

You can try running:

```sh
brew services restart redis
```

to restart the redis service. The Turbo Streams weren't working for me until I also ensured that the `config/cable.yml` was set to use redis in development:

```yml
development:
  adapter: redis
  url: redis://localhost:6379/1
```

I also needed to ensure that redis was uncommented in the Gemfile:

```
# Use Redis adapter to run Action Cable in production
gem "redis", ">= 4.0.1"
```

When I generated a rails application before having redis installed locally, Rails generated different config settings for action cable and also commented out the redis gem. Only after updating those two places was I able to get Turbo Stream updates working in the browser.

## Tour of the Code

The starter point for the code was a tailwind rails scaffold of the `Task` resource with a name and status. We made some modifications to the `TasksController` to enable Turbo Stream responses while leaving the html fallbacks.

### Responding to Turbo Streams in the Controller

```rb
# app/controllers/tasks_controller.rb
class TasksController < ApplicationController
  before_action :set_task, only: %i[ show edit update destroy ]

  # GET /tasks
  def index
    @todo_tasks = Task.todo
    @done_tasks = Task.done
  end

  # GET /tasks/1
  def show
  end

  # GET /tasks/new
  def new
    @task = Task.new
  end

  # GET /tasks/1/edit
  def edit
  end

  # POST /tasks
  def create
    @task = Task.new(task_params)

    respond_to do |f|
      if @task.save
        f.turbo_stream
        f.html { redirect_to @task, notice: "Task was successfully created." }
      else
        f.turbo_stream { render turbo_stream: turbo_stream.replace("#{helpers.dom_id(@task)}_form", partial: "form", locals: { task: @task }) }
        f.html { render :new, status: :unprocessable_entity }
      end
    end
  end

  # PATCH/PUT /tasks/1
  def update
    respond_to do |f|
      if @task.update(task_params)
        f.turbo_stream
        f.html {redirect_to @task, notice: "Task was successfully updated.", status: :see_other}
      else
        f.turbo_stream { render turbo_stream: turbo_stream.replace("#{helpers.dom_id(@task)}_form", partial: "form", locals: { task: @task }) }
        f.html {render :edit, status: :unprocessable_entity}
      end
    end
  end

  # DELETE /tasks/1
  def destroy
    @task.destroy!
    respond_to do |f|
      f.turbo_stream { render turbo_stream: turbo_stream.remove(@task) }
      f.html { redirect_to tasks_url, notice: "Task was successfully destroyed.", status: :see_other }
    end
  end

  private
    # Use callbacks to share common setup or constraints between actions.
    def set_task
      @task = Task.find(params[:id])
    end

    # Only allow a list of trusted parameters through.
    def task_params
      params.require(:task).permit(:name, :status)
    end
end
```

### Adding Turbo Stream Templates

To work with those turbo stream responses, we created two new templates in `app/views/tasks`

```erb
<!-- app/views/tasks/create.turbo_stream.erb -->
<%= turbo_stream.prepend 'tasks' do %>
  <%= render 'task', task: @task %>
<% end %>

<%= turbo_stream.replace 'new_task_form' do %>
  <%= render 'form', task: Task.new %>
<% end %>
```

```erb
<!-- app/views/tasks/update.turbo_stream.erb -->
<%= turbo_stream.remove "#{dom_id(@task)}_li" %>

<% if @task.todo? %>
  <%= turbo_stream.append 'tasks', partial: 'tasks/task', locals: { task: @task } %>
<% else %>
  <%= turbo_stream.prepend 'done-tasks', partial: 'tasks/task', locals: { task: @task } %>
<% end %>
```

### Streaming into the Index Template

Our app is focused on the tasks endpoint where we render the index template.

In the controller, we're sending along two groups of tasks defined in these scopes in the model:

```rb
# app/models/task.rb
class Task < ApplicationRecord
  enum status: { todo: 0, done: 1 }
  validates :name, presence: true
  scope :todo, -> { where(status: "todo").order(:updated_at) }
  scope :done, -> { where(status: "done").order(updated_at: :desc) }
end
```

The values from the scopes are passed to the view template from the controller.

```rb
# app/controllers/tasks_controller.rb
class TasksController < ApplicationController
  # ...
  # GET /tasks
  def index
    @todo_tasks = Task.todo
    @done_tasks = Task.done
  end
  # ...
end
```

And then we render both the form to create a new task and the two task lists within the index template.

```erb
<h2 class="text-2xl mb-2">To Do List</h2>

<div class="px-4 py-6 bg-gray-50 border-gray-100 rounded shadow">
  <div class="mb-10">
    <%= render 'form', task: Task.new %>
  </div>

  <ul class="space-y-3" id="tasks">
    <%= render @todo_tasks %>
  </ul>

  <h2 class="text-2xl mb-2 mt-10">Done</h2>
  <ul class="space-y-3" id="done-tasks">
    <%= render @done_tasks %>
  </ul>
</div>
```

This template relies on both the form partial and the task partial. Let's start by taking a look at the task partial.

```erb
<li id="<%= "#{dom_id(task)}_li" %>">
  <%= turbo_frame_tag dom_id(task) do %>
    <div
      class="flex items-center gap-x-2 group"
      data-controller="task"
    >
      <!-- checkbox to toggle task done/todo -->
      <% if task.done? %>
        <!-- form to edit task as todo -->
      <% else %>
        <!-- form to edit task as done -->
      <% end %>
      <%= link_to task.name, edit_task_path(task), class: task.done? ? 'line-through' : '' %>
      <%= button_to task_path(task), method: :delete, class: 'hidden group-hover:block' do %>
        <span class="text-red-500 font-bold p-1">x</span>
      <% end %>
    </div>
  <% end %>
</li>
```

The `li` for the entire partial is tagged with a `dom_id` that will generate something like: `task_1_li`. We're doing this so that we can remove the element when we do an update (that might change which list it ends up in) or when we delete it.

All of the contents **inside** of the `li` are wrapped in a `turbo_frame_tag` with the `dom_id` that will look like: `task_1`, so we're able to replace the entire contents of each li when we go into edit mode. Clicking on this link:

```erb
<%= link_to task.name, edit_task_path(task), class: task.done? ? 'line-through' : '' %>
```

Will generate a turbo request to the edit template that will replace the contents of the frame in the index template with the matching turbo frame contents in the edit tempate. So, we wrap the edit form in a turbo frame tag in the edit template with a matching id:

```erb
<!-- app/views/tasks/edit.html.erb -->
<!-- ... -->
<%= turbo_frame_tag dom_id(@task) do %>
  <%= render "form", task: @task %>
<% end %>
<!-- ... -->
```

As we can see above, this will render the form partial with an existing saved task, causing the update url to be in the form action.

Let's take a look at the form partial next.

```erb
<%= form_with(model: task, id: "#{dom_id(task)}_form") do |form| %>
  <% if task.errors.any? %>
    <div id="error_explanation" class="bg-red-50 text-red-500 px-3 py-2 rounded-lg mb-3">
      <h2><%= pluralize(task.errors.count, "error") %> prohibited this task from being saved:</h2>

      <ul>
        <% task.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <div class="flex">
    <%= form.text_field :name, class: 'w-full rounded-1 border-gray-300', placeholder: 'Task name', autocomplete: 'off' %>
    <%= form.submit class: '-ml-px px-4 py-2 text-sm rounder-r text-white cursor-pointer bg-indigo-600 hover: bg-indigo-700' %>
  </div>
<% end %>
```

The main thing to note here is the id of the form established at the top:

```erb
form_with(model: task, id: "#{dom_id(task)}_form") ...
```

The main reason we're giving this form an id like this is so can re-render the form with errors using Turbo Stream if we get back a validation error response.

### The Create Action

Let's talk through the create action and how it works. The index action is where we're going to list out the tasks, display a form to persist new tasks, and use turbo to stream them into the template.

```erb
<!-- app/views/tasks/index.html.erb -->
<div class="px-4 py-6 bg-gray-50 border-gray-100 rounded shadow">
  <div class="mb-10">
    <%= render 'form', task: Task.new %>
  </div>

  <ul class="space-y-3" id="tasks">
    <%= render @todo_tasks %>
  </ul>

  <h2 class="text-2xl mb-2 mt-10">Done</h2>
  <ul class="space-y-3" id="done-tasks">
    <%= render @done_tasks %>
  </ul>
</div>
```

When we submit the form to create a new task, the `tasks#create` controller action takes over.

```rb
# app/controllers/tasks_controller.rb
# POST /tasks
def create
  @task = Task.new(task_params)

  respond_to do |f|
    if @task.save
      f.turbo_stream
      f.html { redirect_to @task, notice: "Task was successfully created." }
    else
      f.turbo_stream { render turbo_stream: turbo_stream.replace("#{helpers.dom_id(@task)}_form", partial: "form", locals: { task: @task }) }
      f.html { render :new, status: :unprocessable_entity }
    end
  end
end
```

Upon success, we'll be rendering the corresponding turbo stream template:

```erb
<!-- app/views/tasks/create.html.erb -->
<%= turbo_stream.prepend 'tasks' do %>
  <%= render 'task', task: @task %>
<% end %>

<%= turbo_stream.replace 'new_task_form' do %>
  <%= render 'form', task: Task.new %>
<% end %>
```

This accomplishes two things:

1. It adds the task to the beginning of the ul#tasks list. Note that `'tasks'` argument to `turbo_stream.prepend` matches the `id` attribute of the element that we've defined in the `index.html.erb` template.
2. It replaces the `new_task_form` with a new form (clearing the form values) using the form partial.

Upon failure, we use turbo stream to render the form partial again, passing the `@task` object and its accompanying errors as a local variable so we're able to display the server side validation errors generated by the failed request using the markup specifed in the `app/views/tasks/_form.html.erb` partial.

```rb
f.turbo_stream { render turbo_stream: turbo_stream.replace("#{helpers.dom_id(@task)}_form", partial: "form", locals: { task: @task }) }
```

Similar logic will be used to handle the failure of an `update` as well.

### Updating Tasks

Let's take a look again at `tasks#update` in the controller:

```rb
# app/controllers/task_controller.rb
# ...
def update
  respond_to do |f|
    if @task.update(task_params)
      f.turbo_stream
      f.html {redirect_to @task, notice: "Task was successfully updated.", status: :see_other}
    else
      f.turbo_stream { render turbo_stream: turbo_stream.replace("#{helpers.dom_id(@task)}_form", partial: "form", locals: { task: @task }) }
      f.html {render :edit, status: :unprocessable_entity}
    end
  end
end
# ...
```

Notice that in the else clause, we're using turbo stream to replace the form in the dom with essentially a re-rendered version of itself (as we're rendering the same partial). This allows us to display the form again including the validation errors that we get from the server.

If we are successful in our update, then the `f.turbo_stream` expression runs. In this case, Rails will go and find the `app/views/tasks/update.turbo_stream.erb` template file and run it:

```erb
<!-- app/views/tasks/update.turbo_stream.erb -->
<%= turbo_stream.remove "#{dom_id(@task)}_li" %>

<% if @task.todo? %>
  <%= turbo_stream.append 'tasks', partial: 'tasks/task', locals: { task: @task } %>
<% else %>
  <%= turbo_stream.prepend 'done-tasks', partial: 'tasks/task', locals: { task: @task } %>
<% end %>
```

So, in this case, the Turbo Stream will remove the entire `li` for the task rendered by the task partial and either append it to the list of tasks if it's still in `todo` status, or prepend it to the done-tasks list if it has been marked `done`.

You may notice that this will actually happen whenever an update action is performed–even if we only update the text of the task. This is a limitation of the current approach, if we wish to have different behavior for clicking the checkbox and updating the task name, we could define an instance variable in the update action to identify the case where the status is updated. Only in this case would we want to run the remove and append/prepend logic.

```rb
# PATCH/PUT /tasks/1
def update
  respond_to do |f|
    if @task.update(task_params)
      @status_updated = !!task_params[:status]
      f.turbo_stream { render :update }
      f.html {redirect_to @task, notice: "Task was successfully updated.", status: :see_other}
    else
      # ...
    end
  end
end
```

Then we can update the template with some additional logic. We'll make sure that we only remove the li and place it in the other list (todo/done) if we have updated the status. Otherwise, we'll just replace the li with the re-rendered partial from our successful update:

```erb
<!-- app/views/tasks/update.turbo_stream.erb -->
<% if @status_updated %>
  <%= turbo_stream.remove "#{dom_id(@task)}_li" %>
  <% if @task.todo? %>
    <%= turbo_stream.append 'tasks', partial: 'tasks/task', locals: { task: @task } %>
  <% else %>
    <%= turbo_stream.prepend 'done-tasks', partial: 'tasks/task', locals: { task: @task } %>
  <% end %>
<% else %>
  <%= turbo_stream.replace "#{dom_id(@task)}_li", partial: 'tasks/task', locals: { task: @task } %>
<% end %>
```

You can imagine how we could handle multiple kinds of updates using instance variables in the controller combined with conditional logic in the corresponding turbo stream template.

### Deleting Tasks

Deleting tasks is perhaps the simplest of all when it comes to implementing Hotwire/Turbo. In this case, all we have to do is add a button to do the delete, and then a turbo_stream formatted action!

In this case, within the task partial, we'll display a button with a red x in it that will trigger the request.

```erb
<!-- app/views/tasks/_task.html.erb -->
<!-- ... -->
<%= button_to task_path(task), method: :delete, class: 'hidden group-hover:block' do %>
  <span class="text-red-500 font-bold p-1">x</span>
<% end %>
<!-- ... -->
```

Remember that the entire contents of the `li` in this partial are wrapped in a turbo frame, so Hotwire will only look to replace this part of the page when requests are triggered from within the frame. In this case, we can specify the response to turbo requests in the `tasks#destroy` controller action.

```rb
# app/views/controllers/tasks_controller.rb
# DELETE /tasks/1
def destroy
  @task.destroy!
  respond_to do |f|
    f.turbo_stream { render turbo_stream: turbo_stream.remove(@task) }
    f.html { redirect_to tasks_url, notice: "Task was successfully destroyed.", status: :see_other }
  end
end
```

I noticed when working on this that this remove action will actually only remove the contents of the turbo_frame rendered in the task partial, but the `li` element wrapping it will be kept intact. In order to not leave empty li tags within the lists, it would be better to update the target of the remove action here to include \_li, so that the parent element is removed as well as the contents.

```rb
# app/views/controllers/tasks_controller.rb
# DELETE /tasks/1
def destroy
  @task.destroy!
  respond_to do |f|
    f.turbo_stream { render turbo_stream: turbo_stream.remove("#{helpers.dom_id(@task)}_li") }
    f.html { redirect_to tasks_url, notice: "Task was successfully destroyed.", status: :see_other }
  end
end
```

## Concept Overview

One of the key concepts in Turbo/Hotwire is that Rails will use javascript to replace our html body content with the html body content it fetches from the server upon any links we click or forms we submit.

If we'd like to limit how much of the html body content is replaced by Hotwire, we can use Frames and Streams to draw a boundary around the content that will be replaced with the incoming fetched html.

The ids of elements are of key importance in Hotwire, because they are used to identify the target frame elements for frames within different templates or actions that we can stream to.

## Conclusion

I can't help but feel a bit of nostalgia for my early days learning about Hotwire and Turbo. I remember when I first learned Ruby on Rails having the feeling that it was totally awesome when things were going right, but that I felt totally lost if things went wrong.

To be fair, I feel nowhere near as lost when things go wrong now as I did back then. Still, I have found that the mental model used with Hotwire and Turbo hasn't been the most intuitive to grasp, so I've been checking out the source code lately.

I wanted to put together a bit of a README here as a way of reviewing what I learned in this [LinkedIn Learning course by David Morales](https://www.linkedin.com/learning/hotwire-reactive-ruby-on-rails-applications/hotwire-reactive-ruby-on-rails-applications). I still find that I need a bit of review, but I understand the wiring much better now than I did before I took the course.
