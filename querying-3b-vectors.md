  ​
  Querying 3B Vectors

 ╭──────────────────────────────────────────────────────────────────────╮
 │ https://vickiboykis.com/2026/02/21/querying-3-billion-vectors/       │
 │                                                                      │
 │ Reader Mode                                                          │
 ╰──────────────────────────────────────────────────────────────────────╯

  Recently, I got nerd-sniped by this exchange between Jeff Dean and
  someone trying to query 3 billion vectors. I was curious to see if I
  could implement the optimal map-reduce solution he alludes to in his
  reply.

    ●●● Image image

  A vector is a list/array of floating point numbers of n dimensions,
  where n is the length of the list. The reason you might perform vector
  search is to find words or items that are semantically similar to each
  other, a common pattern in search, recommendations, and generative
  retrieval applications like Cursor which heavily leverage embeddings.

  I started by writing an extremely naive implementation which made the
  following assumptions:

    - we have 3 billion searchable (document) vectors and ~1k query
      vectors (a number I made up)
    - Both of the vector sets are stored on disk in .npy format (simple
      format for storing numpy arrays
    - We’d like to compare each of the query vectors against the larger
      pool of document vectors and return the resulting similarity (dot
      product) for each of the vector combinations.
    - 3k total reference vectors (to see if we could intially run this
      amount before scaling)
    - The vectors are of dimensionality (n) 768, a common dimensionality
      for many models that allow for
    similarity-based embedding queries

    import numpy as np
    from loguru import logger
    import time
    import os
  
    # start with 3_000 vectors to keep things small
    total_vectors_num = 3_000_000_000
    query_vectors_num = 1_000
  
    def generate_random_vectors(num_vectors:int)-> np.array:
        logger.info(f"Generating {num_vectors} vectors...")
        rng = np.random.default_rng()
        vectors = rng.random((num_vectors, 768))
        return vectors
  
    def get_dot_products(vectors_file:np.array, query_vectors:np.array) -> list[np.array]:
        total_products_computed = 0
  
        dot_products = []
        for v in vectors_file:
            for qv in query_vectors:
                dot_product = v @ qv
                dot_products.append(dot_product)
                total_products_computed += 1
                if total_products_computed % 100000 == 0:
                     logger.info(f"Total vectors processed:{total_products_computed}")
  
        return dot_products
  
    # Generate initial vectors and query vectors and write to disk
    doc_vectors = generate_random_vectors(total_vectors_num)
    query_vectors = generate_random_vectors(query_vectors_num)
    np.save('vectors.npy', doc_vectors)
  
    # Load vectors from disk
    logger.info("Loading file from disk...")
    vectors_file = np.load('vectors.npy')
  
    start_time = time.time()
    logger.info("Getting dot products...")
    results =  get_dot_products(vectors_file, query_vectors)
    end_time = time.time()
  
    logger.info(f"Execution time: {end_time - start_time:.4f} seconds")
    logger.info(f"Number of dot products computed: {len(results)}")

  This, predictably, didn’t do so great, even on my M2 Macbook, even at
  3,000 vectors, one million times less than 3 billion embeddings,
  taking 2 seconds.

  Results:

    2025-12-13 17:53:25.675 | INFO     | __main__:generate_random_vectors:9 - Generating 3000 vectors...
    2025-12-13 17:53:25.691 | INFO     | __main__:generate_random_vectors:9 - Generating 1000 vectors...
    2025-12-13 17:53:25.698 | INFO     | __main__:<module>:39 - Loading file from disk...
    2025-12-13 17:53:25.700 | INFO     | __main__:<module>:43 - Getting dot products...
    2025-12-13 17:53:27.688 | INFO     | __main__:get_dot_products:24 - Total vectors processed:3000000
    2025-12-13 17:53:27.688 | INFO     | __main__:<module>:47 - Execution time: 1.9877 seconds
    2025-12-13 17:53:27.688 | INFO     | __main__:<module>:48 - Number of dot products computed: 3000000

  So I vectorized the numpy operation, which made things much faster.

    import numpy as np
    from loguru import logger
    import time
  
    total_vectors_num = 3_000
    query_vectors_num = 1_000
  
    def generate_random_vectors(num_vectors:int)-> np.array:
        logger.info(f"Generating {num_vectors} vectors...")
        rng = np.random.default_rng()
        vectors = rng.random((num_vectors, 768))
        return vectors
  
    def get_dot_products_vectorized(vectors_file:np.array, query_vectors:np.array):
        dot_products = vectors_file @ query_vectors.T
        return dot_products.flatten() # collapse into single dim
  
    # Generate initial vectors and query vectors and write to disk
    ram_vectors = generate_random_vectors(total_vectors_num)
    query_vectors = generate_random_vectors(query_vectors_num)
    np.save('vectors.npy', ram_vectors)
  
    # Load vectors from disk
    logger.info("Loading file from disk...")
    vectors_file = np.load('vectors.npy')
  
    start_time = time.time()
    logger.info("Getting dot products...")
    results =  get_dot_products_vectorized(vectors_file, query_vectors)
    end_time = time.time()
  
    logger.info(f"Execution time: {end_time - start_time:.4f} seconds")
    logger.info(f"Number of dot products computed: {len(results)}")

  At .017 seconds, this was a big improvement!

  Results:

    2025-12-13 17:52:52.810 | INFO     | __main__:generate_random_vectors:9 - Generating 3000 vectors...
    2025-12-13 17:52:52.831 | INFO     | __main__:generate_random_vectors:9 - Generating 1000 vectors...
    2025-12-13 17:52:52.874 | INFO     | __main__:<module>:39 - Loading file from disk...
    2025-12-13 17:52:52.876 | INFO     | __main__:<module>:43 - Getting dot products...
    2025-12-13 17:52:52.887 | INFO     | __main__:<module>:47 - Execution time: 0.0107 seconds
    2025-12-13 17:52:52.887 | INFO     | __main__:<module>:48 - Number of dot products computed: 3000000

  I tried a 3 million sample size with this improvement. This took 12
  seconds.

    2025-12-13 19:39:43.830 | INFO     | __main__:generate_random_vectors:12 - Generating 3000000 vectors...
    2025-12-13 19:39:57.509 | INFO     | __main__:generate_random_vectors:12 - Generating 1000 vectors...
    2025-12-13 19:39:58.978 | INFO     | __main__:<module>:57 - Loading file from disk...
    2025-12-13 19:40:00.131 | INFO     | __main__:<module>:61 - Getting dot products...
    2025-12-13 19:40:12.984 | INFO     | __main__:<module>:65 - Execution time: 12.8491 seconds
    2025-12-13 19:40:12.992 | INFO     | __main__:<module>:66 - Number of dot products computed: 3000000000

  We could also reduce even further by converting the data to float32:

    doc_vectors = generate_random_vectors(total_vectors_num).astype(np.float32)
    query_vectors = generate_random_vectors(query_vectors_num).astype(np.float32)

    2025-12-13 18:13:52.152 | INFO     | __main__:generate_random_vectors:10 - Generating 3000 vectors...
    2025-12-13 18:13:52.168 | INFO     | __main__:generate_random_vectors:10 - Generating 1000 vectors...
    2025-12-13 18:13:52.176 | INFO     | __main__:<module>:55 - Loading file from disk...
    2025-12-13 18:13:52.178 | INFO     | __main__:<module>:59 - Getting dot products...
    2025-12-13 18:13:52.182 | INFO     | __main__:<module>:63 - Execution time: 0.0045 seconds
    2025-12-13 18:13:52.182 | INFO     | __main__:<module>:64 - Number of dot products computed: 3000000

  With these small improvements, we’ve already sped up inference to ~13
  seconds for 3 million vectors, which means for 3 billion, it would
  take 1000x longer, or ~3216 minutes.

    |approach    | query_vectors | doc_vectors   | time     |
    |----------- |---------------|---------------|----------|
    | Naive      | 1,000         | 3,000         | 1.9877s  |
    | Vectorized | 1,000         | 3,000         | 0.0107s  |
    | Vectorized | 1,000         | 3,000,000     | 12.8491s |
    | Np.Float32 | 1,000         | 3,0000        | 0.0045s  |

  When we start to run it to test, however, we run into a different
  problem: OOM. Why? The amount of memory needed to process 3 billion
  objects, each as float32 object that’s 4 bytes in size, would be 8
  million GB.

    
    vectors = rng.random((1, 768)).astype(np.float32)
    print(vectors.nbytes)
    3072
    print(vectors.itemsize)
    4
  
    bytes_per_float32 = 4
    memory_gb = (3000000000 * 1000 * 768 * bytes_per_float32) / (1024**3)
    8583068.84765625 = 8.6 TB

  In order to improve this, we would need to do some heavy lifting of
  the kind Jeff Dean prescribed. First, we could to change the code to
  use generators and batch the comparison operations. We could write
  every n operations to disk, either directly or through memory mapping.
  Or, we could use system-level optimized code calls - we could rewrite
  the code in Rust or C, or use a library like SimSIMD explicitly made
  for similarity comparisons between vectors at scale.

  Before I started on any further optimizations, upon further
  inspection, there were some things about the problem that I realized
  weren’t clear to me: 3 billion vector embeddings queried a few
  thousand times could mean:

    - I have a single query vector, and I query all 3 billion vectors
      once, get the dot product, and get all results
    - I have a single query vector, I query all 3 billion vectors once,
      get the dot product, and return top-k results, which is easier
      because we can do ANN search
      • In this case, do I need to return the two initial vectors also?
        Or just the result?
      • Do I need to re-rank the results by similarity in any way?
    - I have 1,000 query vectors, and I query all 3 billion vectors
      once, and get the dot product of all results
    - Are these vectors already in-memory when we intially start working
      with them or will they always be on-disk? Are we reading them one
      at a time, or streaming them?
    - What kind of machine are we assuming: Are we running this locally?
      What are the specs of the machine? Are we assuming the vectors
      come to us in a specific, optimized format?
      • Do we have GPUs and are we allowed to use them?
    - How big are our embeddings? - this is extremely important and
      could significantly impact our representation, input vector size
      and output results
    - Are we assuming we can compress their representation at all, i.e.
      is compressiong from float64 to float32 tolerable wrt to accuracy?
    - How much time do we have to generate this one-off project? Are we
      sure it’s really a one-off?

  All of these dictate the additional time and resources spent on the
  solution. What I realized is the same thing I’ve seen so many of these
  problems over the years, that the technical solution is no longer the
  hardest one to achieve: the hardest one is nailing down the
  requirements.

