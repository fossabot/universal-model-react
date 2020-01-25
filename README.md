# Universal Model for React

Universal model is a model which can be used with any of following UI frameworks:

- Angular 2+ [universal-model-angular]
- React 16.8+
- Svelte 3+ [universal-model-svelte]
- Vue.js 3+ [universal-model-vue]

## Install

    npm install --save universal-model-react

## Prerequisites for universal-model-react

     "react": "^16.8.0"

## Clean UI Architecture

![alt text](https://github.com/universal-model/universal-model-vue/raw/master/images/mvc.png 'MVC')

- Model-View-Controller (https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller)
- User triggers actions by using view or controller
- Actions are part of model and they manipulate state that is stored
- Actions can use services to interact with external (backend) systems
- State changes trigger view updates
- Selectors select and calculate a transformed version of state that causes view updates
- Views contain NO business logic
- There can be multiple interchangable views that use same part of model
- A new view can be created to represent model differently without any changes to model

## API

Create and export store in store.ts:

    export default createStore(initialState, selectors);

Access store

    const state = store.getState();
    const selectors = store.getSelectors();
    const [state, selectors] = store.getStateAndSelectors();

Use state and selectors in Views
    
     const { subState1, subState2: { prop1 }, subState3: { prop1: myProp } } = store.getState();
     const { selector1, selector2 } = store.getSelectors();

    useState([subState1, prop1, myProp]);
    useSelectors([selector1, selector2]);
    useStateAndSelectors([subState1, prop1, myProp], [selector1, selector2]);

## Clean UI Code directory layout

    - src
      |
      |- common
      |  |- component1
      |  |- component2
      |     .
      |     .
      |     .
      |- componentA
      |- componentB
      |  .
      |  .
      |  .
      |- componentN
      |  |- controller
      |  |- model
      |  |  |- actions
      |  |  |- services
      |  |  |- state
      |  |- view
      |- store

# Example

## View

TodoListView.tsx

    import * as React from 'react';
    import { useEffect } from 'react';
    import store from '../../store/store';
    import { Todo } from '../model/state/initialTodoListState';
    import removeTodo from '../model/actions/removeTodo';
    import fetchTodos from '../model/actions/fetchTodos';
    import todoListController from '../controller/todoListController';
    import toggleIsDoneTodo from '../model/actions/toggleIsDoneTodo';
    import toggleShouldShowOnlyUnDoneTodos from '../model/actions/toggleShouldShowOnlyUnDoneTodos';

    const TodoListView = () => {
      const [{ todosState }, { shownTodos }] = store.getStateAndSelectors();
      store.useStateAndSelectors([todosState], [shownTodos]);

      useEffect(() => {
        // noinspection JSIgnoredPromiseFromCall
        fetchTodos();
        document.addEventListener('keypress', todoListController.handleKeyPress);
        return () => document.removeEventListener('keypress', todoListController.handleKeyPress);
      }, []);

      let todoListContent;

      if (todosState.isFetchingTodos) {
        todoListContent = <div>Fetching todos...</div>;
      } else if (todosState.hasTodosFetchFailure) {
        todoListContent = <div>Failed to fetch todos</div>;
      } else {
        const todoListItems = shownTodos.value.map((todo: Todo, index: number) => (
          <li key={todo.id}>
            <input
              id={todo.name}
              type="checkbox"
              defaultChecked={todo.isDone}
              onChange={() => toggleIsDoneTodo(todo)}
            />
            <label>{todo.name}</label>
            <button onClick={() => removeTodo(todo)}>Remove</button>
          </li>
        ));

        todoListContent = <ul>{todoListItems}</ul>;
      }

      return (
        <div>
          <input
            id="shouldShowOnlyDoneTodos"
            type="checkbox"
            defaultChecked={todosState.shouldShowOnlyUnDoneTodos}
            onChange={toggleShouldShowOnlyUnDoneTodos}
          />
          <label>Show only undone todos</label>
          {todoListContent}
        </div>
      );
    };

    export default TodoListView;

## Controller

todoListController.ts

    import addTodo from "@/todolist/model/actions/addTodo";
    import removeAllTodos from "@/todolist/model/actions/removeAllTodos";

    export default {
      handleKeyPress(keyboardEvent: KeyboardEvent): void {
        if (keyboardEvent.code === 'KeyA' && keyboardEvent.ctrlKey) {
          addTodo();
        } else if (keyboardEvent.code === 'KeyR' && keyboardEvent.ctrlKey) {
          removeAllTodos();
        }
      }
    };
    
## Model

### State

#### Initial state

initialTodoListState.ts

    export interface Todo {
      id: number,
      name: string;
      isDone: boolean;
    }

    export default {
      todos: [] as Todo[],
      shouldShowOnlyUnDoneTodos: false,
      isFetchingTodos: false,
      hasTodosFetchFailure: false
    };

#### State selectors

createTodoListStateSelectors.ts

    import { State } from '@/store/store';
    import { Todo } from '@/todolist/model/state/initialTodoListState';

    const createTodoListStateSelectors = <T extends State>() => ({
      shownTodos: (state: T) =>
        state.todosState.todos.filter(
          (todo: Todo) =>
            (state.todosState.shouldShowOnlyUnDoneTodos && !todo.isDone) ||
            !state.todosState.shouldShowOnlyUnDoneTodos
        )
    });

    export default createTodoListStateSelectors;

### Store

store.ts

    import { createStore } from 'universal-model-react';
    import initialTodoListState from '@/todolist/model/state/initialTodoListState';
    import createTodoListStateSelectors from '@/todolist/model/state/createTodoListStateSelectors';

    const initialState = {
      todosState: initialTodosState,
      otherState: initialOtherState,
      .
      .
    };

    export type State = typeof initialState;

    const selectors = {
      ...createTodosStateSelectors<State>(),
      ...createOtherStateSelectors<State>(),
      .
      .

    };

    export default createStore(initialState, selectors);

### Service

ITodoService.ts

    import { Todo } from '@/todolist/model/state/initialTodoListState';

    export interface ITodoService {
      tryFetchTodos(): Promise<Todo[]>;
    }

FakeTodoService.ts

    import { ITodoService } from './ITodoService';
    import { Todo } from '../state/initialTodoListState';
    import Constants from '../../../Constants';
    
    export default class FakeTodoService implements ITodoService {
      tryFetchTodos(): Promise<Todo[]> {
        return new Promise<Todo[]>((resolve: (todo: Todo[]) => void, reject: () => void) => {
          setTimeout(() => {
            if (Math.random() < 0.95) {
              resolve([
                { id: 1, name: 'first todo', isDone: true },
                { id: 2, name: 'second todo', isDone: false }
              ]);
            } else {
              reject();
            }
          }, Constants.FAKE_SERVICE_LATENCY_IN_MILLIS);
        });
      }
    }

todoService.ts

    import FakeTodoService from "@/todolist/model/services/FakeTodoService";

    export default new FakeTodoService();

### Actions

addTodo.ts

    import store from '../../../store/store';
    
    let id = 3;
    
    export default function addTodo(): void {
      const { todosState } = store.getState();
      todosState.todos.push({ id, name: 'new todo', isDone: false });
      id++;
    }

removeTodo.ts

    import store from '@/store/store';
    import { Todo } from '@/todolist/model/state/initialTodoListState';

    export default function removeTodo(todoToRemove: Todo): void {
      const { todosState } = store.getState();
      todosState.todos = todosState.todos.filter((todo: Todo) => todo !== todoToRemove);
    }

removeAllTodos.ts

    import store from '@/store/store';

    export default function removeAllTodos(): void {
      const { todosState } = store.getState();
      todosState.todos = [];
    }

toggleIsDoneTodo.ts

    import { Todo } from '@/todolist/model/state/initialTodoListState';

    export default function toggleIsDoneTodo(todo: Todo): void {
      todo.isDone = !todo.isDone;
    }

toggleShouldShowOnlyUnDoneTodos.ts

    import store from '@/store/store';

    export default function toggleShouldShowOnlyUnDoneTodos(): void {
      const [{ todosState }] = store.getStateAndSelectors();
      todosState.shouldShowOnlyUnDoneTodos = !todosState.shouldShowOnlyUnDoneTodos;
    }

fetchTodos.ts

    import store from '@/store/store';
    import todoService from '@/todolist/model/services/todoService';

    export default async function fetchTodos(): Promise<void> {
      const { todosState } = store.getState();

      todosState.isFetchingTodos = true;
      todosState.hasTodosFetchFailure = false;

        try {
          todosState.todos = await todoService.tryFetchTodos();
        } catch (error) {
          todosState.hasTodosFetchFailure = true;
        }

        todosState.isFetchingTodos = false;
    }

### Full Example

https://github.com/universal-model/universal-model-react-todo-app

### License

MIT License

[universal-model-angular]: https://github.com/universal-model/universal-model-angular
[universal-model-svelte]: https://github.com/universal-model/universal-model-svelte
[universal-model-vue]: https://github.com/universal-model/universal-model-vue
