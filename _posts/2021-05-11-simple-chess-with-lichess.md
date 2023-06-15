---
layout: post
tags: [lichess, full-stack]
title: Simple chess game with Lichess
---
# [Simple Chess](https://github.com/arthurstomp/simple_chess)

Chess is an awesome ancient board game. Its history trace back 1500 years ago and is still super popular today. Nowadays the two largest platforms that offer the online experience to play chess are [Chess.com](https://chess.com) and [Lichess](https://lichess.org).

`Chess.com` is the largest in terms of player base. Its platform is fast and stable. But it only allow players to play chess in their onw platform. Lichess in other hand, is an open-source and free chess server called [Lila](https://github.com/ornicar/lila). For this nature Lichess not only allows the player to play chess in their hosted platform, but also provides an API so anyone can add a little chess to their site.

Being a developer, my first though when I saw the Lichess API was: Cool! But, now, how can i implement it? So, this post will cover the steps, notes and code i did to implement a chess chess game using Lichess' API.


## What this post covers

* Challenge a Lichess AI bot for a game;
* Listen to the game events and reflect those events in a chessboard using ChessboardJS.
* Play against the AI.

## Authenticate with Lichess

There are 2 ways to which you can authenticate to Lichess server:

* `Personal API access token`
* `OAuth2 authorization code flow`

> Although in Lichess documentation redirects you to [DigitalOcean's OAuth2 documentation](https://www.digitalocean.com/community/tutorials/an-introduction-to-oauth-2#grant-type-authorization-code)
> where it mentions "Implicit authentication" as a possible OAuth2 way of authentication, Lichess authentication(AuthN) does not implement that flow :/

In this blog post I will be using my `Personal API access token` because I won't be manipulating other Lichess users information.

If you want more control over your users it's necessary to implement the `OAuth2 authorization code flow` which requires a back-end.

## Create your Lichess Personal Access Token

* Login into your Lichess account
* Then click in your username on the top right and click on settings;
* In the sidebar(close to the botton) click on `API access token`
* Create a new `Personal API access token` by clicking on the `+` sign button on the top right. Then select the scopes allowed for that token, give it a name and it's done. Now you have access token to play around with Lichess API.

To try your `access token` execute the following commands:

```shell
$ export LICHESS_TOKEN=<your-token-here>
$ curl https://lichess.org/api/account -H "Authorization: Bearer $LICHESS_TOKEN"
```

## Create and react app

Start the application with [create-react-app](https://github.com/facebook/create-react-app) and install some packages

```shellscript
$ create-react-app simple_chess
$ cd simple_chess
$ yarn add @chrisoakman/chessboardjs react-router-dom styled-components react-final-form final-form
```

- [@chrisoakman/chessboardjs](https://chessboardjs.com/): Is a dumb chessboard written in JS. It can easily be connected to Lichess protocol.
- [react-router-dom](https://reactrouter.com/web/guides/quick-start): Allow multiple pages in a single app easily.
- [styled-components](https://styled-components.com/): Helps style the app with components styled with a simple string.
- [react-final-form & final-form](https://final-form.org/react): Helps with forms in general.

## Prepare a context to manage state

Before start coding the app, lets prepare a [Context](https://reactjs.org/docs/context.html) to have an easier access to some states of the application.

> Checkout Paige Niedringhaus' post to learn [How to Use React’s Context API and useContext() Hooks Effectively](https://betterprogramming.pub/how-to-use-reacts-context-api-and-usecontext-hooks-effectively-ed98ad9343b6)


```javascript
// src/context/AppContext.js
import { createContext } from 'react'

const AppContext = createContext({
  accessToken: null,
  setAccessToken: () => {},
  game: null,
  setGame: () => {}
})

export default AppContext
```

For this application we will have two states that will be used globally: the Lichess Persoanl Access Token and the created game. `accessToken`, `setAccessToken()`, `game` and `setGame()` will be used to access the token and the game in any component wrapped by `AppContext.Provider`.

## Basic Application

```javascript
// src/App.jsx

export default function App() {
  const [accessToken, setAccessToken] = useState(null)
  const [game, setGame] = useState(null)

  return (
    <AppWrapper>
      <AppContext.Provider value={{
        accessToken,
        setAccessToken,
        game,
        setGame
      }}>
        <Router>
          <HeaderWrapper>
            <HeaderTitle>Simple Chess</HeaderTitle>
          </HeaderWrapper>
          <ContentWrapper>
            <TokenChecker />
            <Switch>
              <Route
                exact
                path="/"
                component={ConnectLichess} />
              <Route
                exact
                path="/challenge"
                component={Challenge} />
              <Route
                exact
                path="/game"
                component={Game} />
            </Switch>
          </ContentWrapper>
        </Router>
      </AppContext.Provider>
    </AppWrapper>
  );
}
```

At the first 2 lines of `function App()` 2 states are created. Those states will be used as values to the `AppContext.Provider` created earlier.

At the return of `function App()` the `<Router />`, `<Switch />` and `<Routes />` of the application.

## Store Personal Access Token

The `ConnectLichess` component has a single form where the user should past she/he lichess personal access token. That token is then persisted in `localStorage`, set `AppContext`'s `accessToken` and redirects the application to the page to select the AI level.

> Using localStorage to not need to input the access token all the time.

```javascript
// src/components/ConnectLichess.jsx

export default function ConnectLichess(props) {
  const accessTokenContext = useContext(AppContext);

  const persistLichessToken = (values) => {
    window.localStorage.setItem('LICHESS_TOKEN', values.token)
    accessTokenContext.setAccessToken(values.token)
    props.history.push('/challenge')
  }

  return (
    <CenteredContent>
      <FormWrapper>
        <Form
          onSubmit={persistLichessToken}
          render={({ handleSubmit, submitting }) => (
            <form onSubmit={handleSubmit}>
              <Field name="token">
                {({ input, meta }) => (
                  <FormField>
                    <input {...input} type="text" placeholder="Access Token" />
                  </FormField>
                )}
              </Field>
              <div className="buttons">
                <Button type="submit" disabled={submitting}>
                  Start
                </Button>
              </div>
            </form>
          )}
        />
      </FormWrapper>
    </CenteredContent>
  );
}
```

## Challenge AI

Once the user input has input the access token, the application can start using Lichess API.  To encapsulate the functionalities related to Lichess the application has a Lichess class which takes an access token in its constructor and perform the necessary `fetch` request to Lichess API in its other methods

At this point the application, is just trying to challenge an AI player. The `challengeAI` method is responsable for that. It calls for [Lichess API ("Challenge the AI")](https://lichess.org/api#operation/challengeAi) and, if successful, it returns a new game.

```javascript
// src/utils/lichess.js

export default class Lichess {
  constructor(token) {
    this.token = token
  }

  challengeAI(level) {
    const url = `https://lichess.org/api/challenge/ai`
    const params = {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.token}`,
        'Content-Type': 'application/x-www-form-urlencoded'
      },
      body: `level=${level}`
    }

    return fetch(url, params)
  }

  ...

}
```

With the Lichess client ready is time to code the UI. The `Challenge` takes care of it. This component has just a form so the user can select the level of the AI she/he wants to play against. When the form is submitted the component consumes the `accessToken` from `AppContext` and use it to call Lichess's `Challenge the AI`. If the request is successful the returned game is persisted in `AppContext`.

```javascript
const ChallengeForm = props => {
  const { accessToken, setGame, history } = props
  const lichess = new Lichess(accessToken);
  const submit = values => {
    lichess.challengeAI(values.level)
      .then(res => {
        if(res.status === 201) {
          return res.json()
        } else {
          throw new Error('Failed to create challenge')
        }
      })
      .then(data => {
        console.log(data)
        setGame(data)
        history.push('/game')
      })
      .catch(console.error)
  }
  return (
    <FormWrapper>
      <Form
        onSubmit={submit}
        render={({ handleSubmit, submitting, form, pristine }) => (
          <form onSubmit={handleSubmit}>
            <label>AI Level</label>
            <Field name="level">
              {({ input, meta }) => (
                <FormField>
                  <input {...input} type="number" min="1" max="8" />
                </FormField>
              )}
            </Field>
            <div className="buttons">
              <Button type="submit" disabled={submitting}>
                Start Game
              </Button>
            </div>
          </form>
        )}
      />
    </FormWrapper>
  )
}

export default function Challenge(props) {
  const { history } = props
  const appContext = useContext(AppContext)
  const accessToken = appContext.accessToken

  return (
    <CenteredContent>
      <ChallengeWrapper>
        <ChallengeForm
          history={history}
          accessToken={accessToken}
          setGame={appContext.setGame} />
      </ChallengeWrapper>
    </CenteredContent>
  )
}
```

## Lichess to game events

When the Lichess game is created it allows the application to consume a stream of events of that game by request Lichess API's ["Stream Board game state"](https://lichess.org/api#operation/boardGameStream). The game stream events comes formated in [ndjson]() in which each event is a `JSON` object and events are separated by a `new line \n`.

The consumption of this stream is done in Lichess client by looping on the return of the "endless" request for the stream of the game events. The function that loops over the return splits the `ndjson` string and parses the non-blank string to JSON objects, that object is then sent to the callback function and the loops starts over.


```javascript
export default class Lichess {
  constructor(token) {
    this.token = token
  }

  listenGameEvents(gameId, callback) {
    const url = `https://lichess.org/api/board/game/stream/${gameId}`
    const params = {
      method: 'GET',
      headers: {
        'Authorization': `Bearer ${this.token}`
      }
    }
    fetch(url, params)
      .then(res => { this.readStream(res, callback) })
      .catch(err => { console.log(err) })
  }

  readStream(res, callback) {
    const reader = res.body.getReader()
    const read = result => {
      if(result.done) {
        console.log('events: Done.')
        return
      } else {
        const events = this.processNDJSON(result.value)
        events.forEach(callback)
      }
      reader.read().then(read, console.err)
    }
    reader.read().then(read, console.err)
  }

  processNDJSON(value) {
    const decodedValue = new TextDecoder().decode(value)
    return decodedValue
      .split('\n')
      .filter(l => l !== '\n' && l.length !== 0)
      .map(JSON.parse)
  }

  ...
}
```

The `listenGameEvents` can then be used by the `Game` component to start reacting to the events from a Lichess game.

There 2 events that are consumed by the application: `gameState` and `gameFull`. `gameFull` is sent just after the game is created. The application use the `gameFull` state to set the orientation of the table(is the user player white or black) and perform the first move of the game, if the AI play white.

```javascript
// src/components/Game.jsx

const startBoardForGame = (event, board) => {
  if (!event.white.aiLevel) {
    board.orientation('white')
  } else if (!event.black.aiLevel) {
    board.orientation('black')
  } else {
    throw new Error('Current user is not part of the game')
  }
  board.start()
}

const processGameEvent = (game, board, setWinner) => {
  const doProcessGameEvent = (event) => {
    console.log('doProcessGameEvent', event)
    switch(event.type) {
      case 'gameState':
        if(event.winner) {
          return setWinner(event.winner)
        }
        performLichessMove(event, game, board)
        break
      case 'gameFull':
        startBoardForGame(event,board)
        performLichessMove(event.state, game, board)
        break
      default:
    }
  }

  return doProcessGameEvent
}
export default function Game(props) {
  const boardRef = useRef(null)
  const [board, setBoard] = useState(null)
  const [winner, setWinner] = useState(null)
  const appContext = useContext(AppContext)

  const { accessToken, game } = appContext

  useEffect(() => {
    if(game && board) {
      const lichess = new Lichess(accessToken)
      lichess.listenGameEvents(game.id, processGameEvent(game, board, setWinner))
    }
  }, [game, board])

  ...
}
```

## Set up the chessboard

So far the application doesn't have anything in the screen, just some events in the log. To have a chessboard in the page it first need to be set up.

The function that initializes the board, `initBoard`, receives a reference to a DOM Element and a callback function to be called when a piece is dropped on the board.

On initialization, the board receaves a configuration object that contains the following properties:

- `draggable: true`: Allow use to drag pieces;
- `pieceTheme`: To define the location where the board will look for the sprite of pieces;
- `onDragStart`: Prevent player from moving opponent's pieces;

Once initialized the board is returned to be set as the board state of the `Game`component.

```javascript
// src/components/Game.jsx

import '@chrisoakman/chessboardjs/dist/chessboard-1.0.0.js'
import '@chrisoakman/chessboardjs/dist/chessboard-1.0.0.min.css'

const defaultBoardConfig = {
  draggable: true,
  pieceTheme: '/img/chesspieces/wikipedia/{piece}.png',
  onDragStart: (source, piece, position, orientation) => {
    if ((orientation === 'white' && piece.search(/^w/) === -1) ||
      (orientation === 'black' && piece.search(/^b/) === -1)) {
      return false
    }
  }
}

const initBoard = (boardRef, onDrop) => {
  const config = Object.assign(defaultBoardConfig, { onDrop })
  const board = window.Chessboard(boardRef.current, defaultBoardConfig)

  return board
}

export default function Game(props) {
  const boardRef = useRef(null)
  const [board, setBoard] = useState(null)
  const [winner, setWinner] = useState(null)
  const appContext = useContext(AppContext)

  const { accessToken, game } = appContext

  useEffect(() => {
    if(board === null && boardRef) {
      const newBoard = initBoard(boardRef, chessboardMove)
      setBoard(newBoard)
    }
  }, [board, boardRef])

  ...
}
```

## Connect game events and moves to Chessboard

Now that application has a game and board set up, it's time to handle the movements from the player and the AI.

Handle user drag 'n dropping pieces on the board is done by setting the `onDrop` property during the board initialization with `chessboardMove` function. This callback function calls Lichess ["Make a Board move"](https://lichess.org/api#operation/boardGameMove) sending the move made by the drag'n drop of pieces. If the request is successful nothing else needs to be done, but if it fails, `chessboardMove` will return `"snapback"` which will command the board to rollback that last movement.

```javascript
// src/utils/lichess.js

export default class Lichess {
  constructor(token) {
    this.token = token
  }

  move(gameId, move) {
    const url = `https://lichess.org/api/board/game/${gameId}/move/${move}`
    const params = {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.token}`,
      },
    }

    return fetch(url, params)
  }

  ...
}

// src/components/Game.jsx

export default function Game(props) {
  const boardRef = useRef(null)
  const [board, setBoard] = useState(null)
  const [winner, setWinner] = useState(null)
  const appContext = useContext(AppContext)

  const { accessToken, game } = appContext

  const chessboardMove = async (src, dst) => {
    const lichess = new Lichess(accessToken)
    try {
      const res = await lichess.move(game.id, `${src}${dst}`)
      let json = null
      if(res.status === 200) {
        json = await res.json()
        if(!json.ok) { return 'snapback' }
      } else {
        console.error('Invalid move')
        return 'snapback'
      }
    } catch(err) {
      console.error(err)
      return 'snapback'
    }
  }

  useEffect(() => {
    if(board === null && boardRef && game) {
      const newBoard = initBoard(boardRef, game, chessboardMove)
      setBoard(newBoard)
    }
  }, [board, boardRef, game])

  ...
}
```

To handle Lichess AI move the application processes every `gameState` event using the `performLichessMove` function. Every `gameState` event comes with the `moves` property, this property is a string containing all the moves done in the game. That can pouse a problem with the implementation of the players move because the function `performLichessMove` would try to move the players piece twice, once when the player drops the piece and twice when the event of the player move comes back through the game events. To prevent that the `performLichessMove` checkes if the last was performed by the opponent.


```javascript
const performLichessMove = (gameState, board) => {
  const moves = gameState.moves.split(' ')
  const movesSize = moves.length
  const lastMove = moves[movesSize - 1]

  if(!lastMove) { return }

  const chessboardMove = `${lastMove.slice(0,2)}-${lastMove.slice(2,4)}`

  // Player is white and last move is a black move
  if(board.orientation() === 'white' && movesSize % 2 === 0) {
    board.move(chessboardMove)
  }

  // Player is black and last move is a white move
  if(board.orientation() === 'black' && movesSize% 2 === 1) {
    board.move(chessboardMove)
  }
}
```

## The End

This is just an basic way on how to manage chess games using Lichess chess servers.

The code and working example of the topics above can be found at [github.com/arthurstomp/simple_chess](https://github.com/arthurstomp/simple_chess)

Until next time
