```
const POOL_SIZE = 154;
const CHUNK_SIZE = 5;

function getRandomInt(min, max) {
  min = Math.ceil(min);
  max = Math.floor(max);
  return Math.floor(Math.random() * (max - min) + min); // The maximum is exclusive and the minimum is inclusive
}

const promiseGenerator = (num) => {
  const res = [];
  for (let i = 0; i < num; i++) {
    res.push(
      new Promise((resolve, reject) =>
        setTimeout(() => {
            resolve(`Promise.${i} resolved`);
        }, getRandomInt(0, 100))
      )
    );
  }

  return res;
};

function getPoolHandler(makeGenerator, customResHandler) {
  return function () {
    const generator = makeGenerator.apply(this, arguments);

    async function handle(result) {
      if (result.done) {
        return await result.value;
      }

      return result.value.then(
        function (res) {
          customResHandler(res);
          return handle(generator.next(res));
        },
        function (err) {
          return handle(generator.throw(err));
        }
      );
    }

    try {
      return handle(generator.next());
    } catch (ex) {
      return Promise.reject(ex);
    }
  };
}

function* exec(p, chunkSize) {
  let t = [];

  for (let k = 0; k < p.length; k++) {
    if (t.length < chunkSize) {
      console.log(`Adding to temp pool ${k}`);
      t.push(p[k]);
    }

    if (t.length >= chunkSize || k >= p.length - 1) {
      console.log(`Flush temp pool of ${t.length}`);

      yield Promise.all(t);

      t = [];
    }
  }
}

const handlePromisePool = getPoolHandler(exec, (res) => console.log(res));

const pool = promiseGenerator(POOL_SIZE);

handlePromisePool(pool, CHUNK_SIZE);
```
