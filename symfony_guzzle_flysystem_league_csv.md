# Download, process and store CSV files using Symfony
## Or how you can use Guzzle, Flysystem, League CSV and Doctrine with Symfony

Many developers have faced the situation in which they need to access to a customer's ftp server and download some CSV files in order to process and store the content into the database.

In this article, I will show you how you can do that using Symfony with some useful bundles in order to achieve our goal.

This is the list of bundles I intentend to use:

1. Symfony GuzzleBundle: it's a bundle that integrates Guzzle into Symfony. 
https://github.com/8p/EightPointsGuzzleBundle   
http://docs.guzzlephp.org/en/stable/

2. Flysystem bundle: This bundle integrates Flysystem into Symfony.
https://github.com/thephpleague/flysystem-bundle   
https://flysystem.thephpleague.com/docs/

3. League/CSV: With this bundle you can deal with CSV data without complexity.
https://csv.thephpleague.com/


Install and configure bundle:
=============================

## Configure Guzzle bundle:

I am going to use Guzzle to access the ftp server where are my CSV files located.

`composer require eightpoints/guzzle-bundle`

For Symfony versions < 4.0, load the bundle in AppKernel:

`new EightPoints\Bundle\GuzzleBundle\EightPointsGuzzleBundle`

Now, I will show you how to configure the bundle. In **config.yml** file, add the following lines of code:

```
# config.yml

# Guzzle configuration
eight_points_guzzle:
    logging: "%kernel.debug%"

    clients:
        ftp_server:
            base_url: "%ftp_server.base_url%"
            options:
                auth:
                    - '%ftp_server.username%'
                    - '%ftp_server.password%'
                headers:
                    Accept: "text/html"
                    Content-Type: "text/html"
```

Don't forget to set the necessary parameters in **parameters.yml**:

```
# parameters.yml
ftp_server.base_url: 'your_ftp_server_url'
    ftp_server.username: 'username'
    ftp_server.password: 'password'
```

## Configure Flysystem:

Flysystem is used as an abstraction for the filesystem, whether you are on local or in cloud, you can manipulate your filesystem whithout losing your mind.

To install the bundle use composer require:

`composer require league/flysystem-bundle`

I am going to use my local filesystem to store the CSV files into my disk. However, if you want to use some cloud provider, you can get more details about this topic from Flysystem docs. The link to the docs is further above.

After installing, we need to add configuration code in **config.yml** file:

```
# config.yml

# Flysystem configuration
oneup_flysystem:
    adapters:
        csvfiles_adapter:
            local:
                directory: '%media.csvfiles.dir%'

    filesystems:
        feedfiles:
            adapter: csvfiles_adapter
```

## Configure League/CSV:

The last part in configuration is to install the League/CSV bundle.

`composer require league/csv`

And we are done with the configuration.

Code implementation:
====================

Before, starting our code implementation, let me tell you a little bit about the steps we are going to follow and how to reach our goal.

As I mentioned in the introduction, our main goal is to access to a ftp server where our CSV files are located in order to process them.

Here are the steps I am going to follow:

1. Connect through Guzzle to the ftp server.
2. Check if the files we are looking for exist.
3. Download the CSV file to our local filesystem.
4. Process the CSV files and prepare the content.
5. Store the content of the file into our DB.


## CSV file structure:

Let's suppose in this example that we are going to process an Article CSV file with these data:

- Id.
- Name.
- Price.
- Expiration date.

So the first thing to do, is to create a Doctrine entity in our project and to create a table in our database that will hold the CSV file data.

### Create Article entity:

```
<?php
# Article.php

namespace AppBundle\Entity;

class Article
{
    /**
     * @var int
     */
    private $id;

    /**
     * @var string
     */
    private $name;

    /**
     * @var float
     */
    private $price;

    /**
     * @var \DateTime
     */
    private $expirationDate;

    /**
     * @return int
     */
    public function getId(): int
    {
        return $this->id;
    }

    /**
     * @param int $id
     */
    public function setId(int $id): void
    {
        $this->id = $id;
    }

    /**
     * @return string
     */
    public function getName(): string
    {
        return $this->name;
    }

    /**
     * @param string $name
     */
    public function setName(string $name): void
    {
        $this->name = $name;
    }

    /**
     * @return float
     */
    public function getPrice(): float
    {
        return $this->price;
    }

    /**
     * @param float $price
     */
    public function setPrice(float $price): void
    {
        $this->price = $price;
    }

    /**
     * @return \DateTime
     */
    public function getExpirationDate(): \DateTime
    {
        return $this->expirationDate;
    }

    /**
     * @param \DateTime $expirationDate
     */
    public function setExpirationDate(\DateTime $expirationDate): void
    {
        $this->expirationDate = $expirationDate;
    }
}
```

Since I am not using annotations in this example, I need to define mapping rules in an orm file:

```
# Article.orm.yml
AppBundle\Entity\Article:
  type: entity
  repositoryClass: AppBundle\Repository\ArticleRepository
  table: article
  id:
    id:
      type: integer
      column: id
      options:
        unsigned: true
  fields:
    name:
      type: string
      length: 255
      nullable: true
      column: name
      options:
        fixed: false
    price:
      type: float
      nullable: true
      precision: 20
      scale: 2
      column: price
      options:
        default: 0.00
    expirationDate:
      type: datetime
      nullable: false
      column: expiration_date
```
   
### Create the command:

In this example I will use a Symfony Console Command to run the necessary code.
So, let's create our command:

```
<?php

namespace AppBundle\Command;

use AppBundle\Services\{CsvFilesManager};
use GuzzleHttp\Client;
use GuzzleHttp\Exception\{GuzzleException};
use League\Csv\{Reader, Exception, Writer};
use League\Flysystem\{Filesystem, FileNotFoundException};
use Symfony\Component\Console\{Command\Command,
    Input\InputInterface,
    Input\InputOption,
    Output\OutputInterface,
    Style\SymfonyStyle};
use Psr\Log\LoggerInterface;
use Iterator;

/**
 * Class CsvFilesCommand
 *
 * @command php bin/console csv:files -f <filename>
 *
 * @package AppBundle\Command
 */
class CsvFilesCommand extends Command
{
    const BATCH_SIZE = 200;
    const ARTICLE_COLUMN_NAMES = [
        'id',
        'name',
        'price',
        'expiration_date'
    ];

    /**
     * @var CsvFilesManager
     */
    protected $csvFilesManager;

    /**
     * @var ArticleManager
     */
    protected $articleManager;

    /**
     * @var Filesystem
     */
    protected $filesystem;

    /**
     * @var Client
     */
    protected $client;

    /**
     * @var string
     */
    protected $baseUrl;

    /**
     * @var string
     */
    protected $username;

    /**
     * @var string
     */
    protected $password;

    /**
     * @var SymfonyStyle
     */
    protected $symfonyStyle;

    /**
     * @var LoggerInterface
     */
    protected $logger;

    /**
     * CSVFilesCommand constructor.
     * @param Client $client
     * @param string $baseUrl
     * @param string $username
     * @param string $password
     * @param CsvFilesManager $csvFilesManager
     * @param ArticleManager $articleManager
     * @param Filesystem $filesystem
     * @param SymfonyStyle $symfonyStyle
     * @param LoggerInterface $logger
     */
    public function __construct(
        Client $client,
        string $baseUrl,
        string $username,
        string $password,
        CsvFilesManager $csvFilesManager,
        ArticleManager $articleManager,
        Filesystem $filesystem,
        SymfonyStyle $symfonyStyle,
        LoggerInterface $logger
    ) {
        parent::__construct();

        $this->client = $client;
        $this->baseUrl = $baseUrl;
        $this->username = $username;
        $this->password = $password;
        $this->csvFilesManager = $csvFilesManager;
        $this->articleManager = $articleManager;
        $this->filesystem = $filesystem;
        $this->symfonyStyle = $symfonyStyle;
        $this->logger = $logger;
    }

    /**
     * Configure command
     */
    public function configure()
    {
        $this
            ->setName('csv:files')
            ->setDescription('Download and process CSV files from ftp server')
            ->->addOption('file', 'f', InputOption::VALUE_REQUIRED, 'Csv file name to download and process')
        ;

    }

    /**
     * @param InputInterface $input
     * @param OutputInterface $output
     * @return int|void|null
     * @throws GuzzleException
     * @throws FileNotFoundException
     */
    public function execute(InputInterface $input, OutputInterface $output)
    {
        $this->symfonyStyle->title("CSV file processing just started ...");
        $csvFileName = $input->getOption('file');

        if (empty($csvFileName)) {
            $this->symfonyStyle->error(sprintf("Filename option is missing, it's required to store your CSV file!"));
            exit();
        }

        $this->symfonyStyle->progressStart();
        $this->symfonyStyle->newLine(2);
        $start = microtime(true);

        $this->executeCsvFeed();

        $this->symfonyStyle->newLine();
        $this->symfonyStyle->success(
            sprintf(
                "CSV file processing finished in %d!",
                round(microtime(true) - $start, 4)
            )
        );
        $this->symfonyStyle->progressFinish();
    }

    /**
     * @throws FileNotFoundException
     * @throws GuzzleException
     */
    public function executeCsvFeed() {
        $this->symfonyStyle->writeln("CSV feed execution just started ...");

        $fileName = $this->csvFilesManager->getFileContent(
            $this->baseUrl,
            $csvFileName,
            'GET',
            $this->username,
            $this->password
        );

        if (!empty($fileName)) {
            $file = $this->csvFilesManager->getFile(
                $this->baseUrl,
                $fileName,
                'GET',
                $this->username,
                $this->password
            );

            if (!empty($file)) {
                $records = $this->processCsvFile($file);
                $this->storeCsvFeed($records);
            }
            else {
                $this->logger->error(
                    sprintf("Failed to process feed file: %s - id: %d\n",
                        $fileName
                    )
                );
            }

            $this->symfonyStyle->success("Csv Feed execution finished!");
        }
        else {
            $this->symfonyStyle->error("No CSV files found!");
            $this->logger->error("CSV feed Execution failed: No files found!");
        }
    }

    /**
     * Store Csv Feed.
     *
     * @param Iterator $records
     * @return int|null
     */
    public function storeCsvFeed(Iterator $records) {
        $this->symfonyStyle->text('Storing CSV Feed begins...');
        return $this->articleManager->storeCsvFeed($records);
    }

    /**
     * @param string $file
     * @return Iterator|null
     * @throws FileNotFoundException
     */
    public function processCsvFile(string $file) {
        try {
            $this->symfonyStyle->text(sprintf("\nBegin processing CSV file: %s\n", $file));

            $fileContent = gzdecode($this->filesystem->read($file));
            $csvFile = Reader::createFromString($fileContent);
            $csvFile->setDelimiter(";");
            $csvFile->setHeaderOffset(0);

            $this->symfonyStyle->text(sprintf("\nEnd processing CSV file: %s\n", $file));

            return $csvFile->getRecords();
        } catch (Exception $e) {
            $this->symfonyStyle->error($e->getMessage());
        }

        return null;
    }
}
```

Now after the command implmentation, we will need to implement 2 class managers, CsvFeedManager and ArticleManager:

```
<?php

namespace AppBundle\Services;

use AppBundle\Command\CsvFilesCommand;
use AppBundle\Entity\{Article};
use Doctrine\ORM\EntityManagerInterface;
use GuzzleHttp\{Client, Psr7};
use GuzzleHttp\Exception\{GuzzleException, RequestException};
use League\Flysystem\Filesystem;
use Symfony\Component\Console\Style\SymfonyStyle;
use Psr\Log\LoggerInterface;
use Exception;
use DateTime;

/**
 * Class csvFilesManager
 * @package AppBundle\Services
 */
class csvFilesManager
{
    /**
     * @var Client
     */
    private $client;

    /**
     * @var Filesystem
     */
    private $filesystem;

    /**
     * @var SymfonyStyle
     */
    private $symfonyStyle;

    /**
     * @var LoggerInterface
     */
    private $logger;

    /**
     * CsvFilesManager constructor.
     * @param Client $client
     * @param Filesystem $filesystem
     * @param SymfonyStyle $symfonyStyle
     * @param LoggerInterface $logger
     */
    public function __construct(
        Client $client,
        Filesystem $filesystem,
        SymfonyStyle $symfonyStyle,
        LoggerInterface $logger
    ) {
        $this->client = $client;
        $this->filesystem = $filesystem;
        $this->symfonyStyle = $symfonyStyle;
        $this->logger = $logger;
    }

    /**
     * @param string $baseUrl
     * @param string $fileName
     * @param string $method
     * @param string $username
     * @param string $password
     * @return mixed|string|null
     * @throws GuzzleException
     */
    public function getFile(string $baseUrl, string $fileName, string $method, string $username, string $password) {
        try {
            $filePath = $baseUrl . $fileName;
            $fileContent = tmpfile();
            $response = $this->client->request($method, $filePath, [
                'auth' => [$username, $password],
                'headers'        => ['Accept-Encoding' => 'gzip'],
                'decode_content' => true,
                'sink' => $fileContent
            ]);
            rewind($fileContent);
            $fileName = str_replace('.gz', '', $fileName);
            $this->filesystem->put($fileName, $fileContent);
            fclose($fileContent);

            return $fileName;
        } catch(RequestException $e) {
            echo Psr7\str($e->getRequest());
            if ($e->hasResponse()) {
                echo Psr7\str($e->getResponse());
            }
        }

        return null;
    }

    /**
     * @param string $baseUrl
     * @param string $fileName
     * @param string $method
     * @param $username
     * @param $password
     * @return string
     * @throws GuzzleException
     */
    public function getFileContent(string $baseUrl, string $fileName, string $method, $username, $password): string {
        $filePath = $baseUrl . $fileName;

        try {
            $this->symfonyStyle->writeln("Fetching for files ...");
            $response = $this->client->request($method, $filePath, ['auth' => [$username, $password]]);
            return $response->getBody()->getContents();
        } catch (RequestException $e) {
            $this->symfonyStyle->error(sprintf("%s", Psr7\str($e->getRequest())));
            $this->logger->error(sprintf("%s", Psr7\str($e->getRequest())));
            if ($e->hasResponse()) {
                $this->symfonyStyle->error(sprintf("%s", Psr7\str($e->getResponse())));
                $this->logger->error(sprintf("%s", Psr7\str($e->getResponse())));
            }
        }

        return null;
    }
}
```

```
<?php


namespace AppBundle\Services;

use AppBundle\Command\CsvFilesCommand;
use AppBundle\Entity\Article;
use Doctrine\ORM\EntityManagerInterface;
use Psr\Log\LoggerInterface;
use Symfony\Component\Console\Style\SymfonyStyle;
use Iterator;

/**
 * Class ArticleManager
 * @package AppBundle\Services
 */
class ArticleManager
{
    /**
     * @var EntityManagerInterface
     */
    private $entityManager;

    /**
     * @var SymfonyStyle
     */
    private $symfonyStyle;

    /**
     * @var LoggerInterface
     */
    private $logger;

    /**
     * ArticleManager constructor.
     * @param EntityManagerInterface $entityManager
     * @param SymfonyStyle $symfonyStyle
     * @param LoggerInterface $logger
     */
    public function __construct(
        EntityManagerInterface $entityManager,
        SymfonyStyle $symfonyStyle,
        LoggerInterface $logger
    ) {
        $this->entityManager = $entityManager;
        $this->symfonyStyle = $symfonyStyle;
        $this->logger = $logger;
    }

    /**
     * Store CSV Feed.
     *
     * @param Iterator $records
     * @return int
     */
    public function storeCsvFeed(Iterator $records) {
        $count = 0;

        if (!empty($records)) {
            foreach ($records as $record) {
                $article = new Article();
                $article->setId($record["id"]);
                $article->setName($record["name"]);
                $article->setPrice($record["price"]);
                $article->setExpirationDate($record["expiration_date"]);
                $this->entityManager->persist($article);

                $count++;

                if (($count % CsvFilesCommand::BATCH_SIZE) === 0) {
                    $this->entityManager->flush();
                    $this->entityManager->clear();
                }
            }
            $this->entityManager->flush();
            $this->entityManager->clear();
        }

        return $count;
    }
}
```

Your command now is ready to download, process and store CSV file content into the database.

Just execute in you console (you should be placed in project's directory root):

`php bin/console csv:files -f  <filename>`