# bitstamp-kiss

> is a Bitstamp API v2 wrapper with the joy of kiss literate programming

[![JavaScript Style Guide](https://img.shields.io/badge/code_style-standard-brightgreen.svg)](https://standardjs.com)

## Annotated source

### Dependecies

```javascript
const crypto = require('crypto')
const https = require('https')
const querystring = require('querystring')
```

### Enviroment

```javascript
const BITSTAMP_APIKEY = process.env.BITSTAMP_APIKEY
const BITSTAMP_APISECRET = process.env.BITSTAMP_APISECRET
const BITSTAMP_CUSTOMERID = process.env.BITSTAMP_CUSTOMERID
```

### Utils

#### coerceTick

Convert raw tick `Object<String>` into numeric values.

```javascript
function coerceTick (tick) {
  return {
    high: parseFloat(tick.high),
    last: parseFloat(tick.last),
    timestamp: parseInt(tick.timestamp),
    bid: parseFloat(tick.bid),
    vwap: parseFloat(tick.vwap),
    volume: parseFloat(tick.volume),
    low: parseFloat(tick.low),
    ask: parseFloat(tick.ask),
    open: parseFloat(tick.open)
  }
}
```

#### getNonce

Get a unique progressive value. Current UTC timestamp is used, as usual.
It is also to return value in milliseconds, to make the nonce unique.

```javascript
function getNonce () {
  const now = new Date()

  const nonce = new Date(now.getUTCFullYear(), now.getUTCMonth(), now.getUTCDate(), now.getUTCHours(), now.getUTCMinutes(), now.getUTCSeconds()).getTime()

  return nonce + now.getUTCMilliseconds()
}
```

#### getSignature

```javascript
function getSignature (nonce) {
  const hmac = crypto.createHmac('sha256', BITSTAMP_APISECRET)

  const message = nonce + BITSTAMP_CUSTOMERID + BITSTAMP_APIKEY

  hmac.update(message)

  const signature = hmac.digest('hex').toUpperCase()

  return signature
}
```

#### limitTo8Decimals

Check value to avoid Bistamp API error:

> Ensure that there are no more than 8 decimal places.

```javascript
function limitTo8Decimals (value) {
  const decimals = value.toString().split('.')[1]

  if (decimals && decimals.length > 8) {
    return value.toFixed(8)
  } else {
    return value
  }
}
```

### Public API

#### publicRequest

```javascript
function publicRequest (path, next) {
  https.get(`https://www.bitstamp.net/api${path}`, (response) => {
    const statusCode = response.statusCode

    if (statusCode !== 200) {
      const error = new Error(`Request failed with ${statusCode}`)

      response.resume()

      next(error)
    }

    response.setEncoding('utf8')

    let responseJSON = ''

    response.on('data', (chunk) => {
      responseJSON += chunk
    })

    response.on('end', () => {
      const responseData = JSON.parse(responseJSON)

      if (responseData.status === 'error') {
        const error = new Error(responseData.reason)
        error.code = responseData.code
        next(error)
      } else {
        next(null, responseData)
      }
    })
  })
}
```

#### orderBook

Returns a JSON dictionary like the ticker call, with the calculated values being from within an hour.

```javascript
function orderBook (currencyPair, next) {
  const path = `/v2/order_book/${currencyPair}/`

  publicRequest(path, next)
}

exports.orderBook = orderBook
```

#### ticker

```javascript
/**
 * Returns data for the currency pair.
 *
 * @returns {Object} tick
 * @returns {String} tick.last Last currency price.
 * @returns {String} tick.high Last 24 hours price high.
 * @returns {String} tick.low Last 24 hours price low.
 * @returns {String} tick.vwap Last 24 hours [volume weighted average price](https://en.wikipedia.org/wiki/Volume-weighted_average_price).
 * @returns {String} tick.volume Last 24 hours volume.
 * @returns {String} tick.bid Highest buy order.
 * @returns {String} tick.ask Lowest sell order.
 * @returns {String} tick.timestamp Unix timestamp date and time.
 * @returns {String} tick.open First price of the day.
 */

function ticker (currencyPair, next) {
  const path = `/v2/ticker/${currencyPair}/`

  publicRequest(path, (err, data) => {
    next(err, coerceTick(data))
  })
}

exports.ticker = ticker
```

#### transaction

```javascript
/**
 * @params {String} time The time interval from which we want the transactions to be returned. Possible values are minute, hour (default) or day.
 */

function transactions (currencyPair, time, next) {
  const path = `/v2/transactions/${currencyPair}/?time=${time}`

  publicRequest(path, next)
}

exports.transactions = transactions
```

### Private API

#### privateRequest

```javascript
function privateRequest (path, params, next) {
  const nonce = getNonce()
  const signature = getSignature(nonce)

  const requestData = querystring.stringify(Object.assign({}, params, {
    key: BITSTAMP_APIKEY,
    signature,
    nonce
  }))

  const requestOptions = {
    hostname: 'www.bitstamp.net',
    port: 443,
    path: `/api/${path}`,
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded',
      'Content-Length': requestData.length,
      'Accept': 'application/json'
    }
  }

  const request = https.request(requestOptions, (response) => {
    response.setEncoding('utf8')

    let responseJSON = ''

    response.on('data', (chunk) => {
      responseJSON += chunk
    })

    response.on('end', () => {
      const responseData = JSON.parse(responseJSON)

      if (responseData.status === 'error') {
        const error = new Error(responseData.reason)
        error.code = responseData.code
        next(error)
      } else {
        next(null, responseData)
      }
    })
  })

  request.on('error', error => {
    next(error)
  })

  request.write(requestData)

  request.end()
}
```

#### accountBalance

```javascript
function accountBalance (next) {
  privateRequest('/v2/balance/', {}, next)
}

exports.accountBalance = accountBalance
```

#### allOpenOrders

```javascript
function allOpenOrders (currencyPair, next) {
  privateRequest('/v2/open_orders/all/', {}, next)
}

exports.allOpenOrders = allOpenOrders
```

#### buyMarketOrder

```javascript
function buyMarketOrder (currencyPair, amount, next) {
  const params = {
    amount: limitTo8Decimals(amount)
  }

  privateRequest(`/v2/buy/market/${currencyPair}/`, params, next)
}

exports.buyMarketOrder = buyMarketOrder
```

#### openOrders

```javascript
function openOrders (currencyPair, next) {
  privateRequest(`/v2/open_orders/${currencyPair}`, {}, next)
}

exports.openOrders = openOrders
```

#### sellMarketOrder

```javascript
function sellMarketOrder (currencyPair, amount, next) {
  const params = {
    amount: limitTo8Decimals(amount)
  }

  privateRequest(`/v2/sell/market/${currencyPair}/`, params, next)
}

exports.sellMarketOrder = sellMarketOrder
```

#### userTransactions

```javascript
/**
 * Returns a descending list of transactions, represented as dictionaries.
 *
 * @param {Number} offset Skip that many transactions before returning results (default: 0).
 * @param {Number} limit Limit result to that many transactions (default: 100; maximum: 1000).
 * @param {Number} sort Sorting by date and time: asc - ascending; desc - descending (default: desc).
 */

function userTransactions (currencyPair, offset, limit, sort, next) {
  const params = {
    offset, limit, sort
  }

  privateRequest(`/v2/user_transactions/${currencyPair}/`, params, next)
}

exports.userTransactions = userTransactions
```

## License

[MIT](http://g14n.info/mit-license/)

