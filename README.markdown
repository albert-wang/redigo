Redigo
======

[![Build Status](https://travis-ci.org/garyburd/redigo.svg?branch=master)](https://travis-ci.org/garyburd/redigo)
[![GoDoc](https://godoc.org/github.com/garyburd/redigo/redis?status.svg)](https://godoc.org/github.com/garyburd/redigo/redis)

Redigo is a [Go](http://golang.org/) client for the [Redis](http://redis.io/) database.

Features
-------

* A [Print-like](http://godoc.org/github.com/garyburd/redigo/redis#hdr-Executing_Commands) API with support for all Redis commands.
* [Pipelining](http://godoc.org/github.com/garyburd/redigo/redis#hdr-Pipelining), including pipelined transactions.
* [Publish/Subscribe](http://godoc.org/github.com/garyburd/redigo/redis#hdr-Publish_and_Subscribe).
* [Connection pooling](http://godoc.org/github.com/garyburd/redigo/redis#Pool).
* [Script helper type](http://godoc.org/github.com/garyburd/redigo/redis#Script) with optimistic use of EVALSHA.
* [Helper functions](http://godoc.org/github.com/garyburd/redigo/redis#hdr-Reply_Helpers) for working with command replies.

Documentation
-------------

- [API Reference](http://godoc.org/github.com/garyburd/redigo/redis)
- [FAQ](https://github.com/garyburd/redigo/wiki/FAQ)

Installation
------------

Install Redigo using the "go get" command:

    go get github.com/garyburd/redigo/redis

The Go distribution is Redigo's only dependency.

Related Projects
----------------

- [rafaeljusto/redigomock](https://godoc.org/github.com/rafaeljusto/redigomock) - A mock library for Redigo.
- [chasex/redis-go-cluster](https://github.com/chasex/redis-go-cluster) - A Redis cluster client implementation.

Contributing
------------

Gary is looking for someone to take over maintenance of this project. If you are interested, contact Gary at the email address listed on his GitHub profile page.

PRs for major features will not be accepted until a new maintainer is found.  Bug reports and PRs for bug fixes are welcome.

License
-------

Redigo is available under the [Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0.html).

Sentinel Aware Pool Sample
--------------------------
```
package main

import (
	"fmt"
	"log"
	"time"

	"github.com/albert-wang/redigo/redis"
)

func main() {
	log.SetFlags(log.LstdFlags | log.Lshortfile)

	sentinels := []string{"127.0.0.1:17000", "127.0.0.1:17001", "127.0.0.1:17002"}
	sent, err := redis.NewSentinelClient("tcp", sentinels, nil, time.Second, time.Second, time.Second)
	if err != nil {
		log.Print(err)
	}

	pool := &redis.SentinelAwarePool{
		Pool: redis.Pool{
			MaxIdle:     10,
			IdleTimeout: 240 * time.Second,
			TestOnBorrow: func(c redis.Conn, t time.Time) error {
				if !redis.TestRole(c, "master") {
					return fmt.Errorf("Failed role check")
				}

				return nil
			},
		},

		TestOnReturn: func(c redis.Conn) error {
			if !redis.TestRole(c, "master") {
				return fmt.Errorf("Failed role check")
			}

			return nil
		},
	}

	pool.Dial = func() (redis.Conn, error) {
		addr, err := sent.QueryConfForMaster("production")
		if err != nil {
			return nil, err
		}

		log.Print("Connecting to ", addr)
		pool.UpdateMaster(addr)

		conn, err := redis.Dial("tcp", addr)
		if err != nil {
			return nil, err
		}

		if !redis.TestRole(conn, "master") {
			return nil, fmt.Errorf("Failed role check")
		}

		return conn, err
	}

	for v := 0; v < 100000000; v++ {

		conn := pool.Get()
		defer conn.Close()

		_, err := conn.Do("SET", fmt.Sprintf("foo%d", v), v)
		if err != nil {
			log.Print(err)
			time.Sleep(time.Second / 2)
		} else {
			value, err := redis.Int(conn.Do("GET", fmt.Sprintf("foo%d", v)))
			if err != nil {
				log.Print(err)
			} else {
				if value != v {
					log.Print("INCONSISTENT: ", v, "!=", value)
				}
			}
			time.Sleep(time.Second / 2)
		}
	}
}
```