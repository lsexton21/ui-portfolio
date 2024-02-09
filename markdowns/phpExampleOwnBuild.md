## My Own PHP API Sample- With SLIM 4 AND Doctrine 3 (NEWEST VERSIONS!)

This is a sample of an entity, service, controller, route stack I personally built when building a new API to manage our signup app.  This uses the newest version of SLIM (v4) and Doctrine(v3) for ORM.
Key differences here that are not in the other PHP example: strict typing, using attributes for Doctrine's ORM, and more comments (mainly for me as it was my first build).

---

#### The Routes (pretty basic since there are only a few for this app, and there is not auth middleware needed)
```
<?php

declare(strict_types=1);

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Slim\App;
use App\Application\Controllers\SignupController;


return function (App $app) {
    // CORS is being handled by AWS API Gateway
    $app->options('/{routes:.*}', function (ServerRequestInterface $request, ResponseInterface $response) {
        return $response;
    });

    $app->get('/signups', SignupController::class . ':getAllSignups');
    
    $app->post('/init-signup', SignupController::class . ':initSignup');   

    $app->post('/reinit-captcha', SignupController::class . ':reinitCaptcha');

    $app->post('/send-sms', SignupController::class . ':sendSms');

    $app->post('/validate', SignupController::class . ':validate');

    $app->post('/complete-signup', SignupController::class . ':completeSignup');

    $app->get('/deactivate-expired-signups', SignupController::class . ':deactivateExpiredSignups');
};
```


#### The Signup Controller
```
<?php

namespace App\Application\Controllers;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface as RequestInterface;
use Psr\Log\LoggerInterface;
use App\Application\Validator;
use App\Application\ParamValidator;
use App\Application\PostParams;
use App\Application\Response;
use App\Application\Database;
use App\Application\Skyetel\SkyetelUtil;
use App\Application\CustomException;
use App\Application\Email;
use App\Application\Services\SignupService;
use App\Application\Services\CaptchaService;
use Doctrine\ORM\EntityManager;
use App\Application\Recaptcha;

class SignupController
{
    public function __construct(private Database $db, private LoggerInterface $log, private EntityManager $entityManager)
    {
        $this->db = $db;
        $this->log = $log;
        $this->entityManager = $entityManager;
    }



    public function getAllSignups(RequestInterface $request, ResponseInterface $response, array $args)
    {
        $signupService = new SignupService($this->db, $this->log, $this->entityManager);
        $signups = $signupService->getAllSignups();

        return Response::success($response, $signups, 200);
    }



    public function initSignup(RequestInterface $request, ResponseInterface $response, array $args)
    {
        // get params and form validations
        $params = (array)$request->getParsedBody();
        $this->log->info("Signup POST data: " . print_r($params, true));

        PostParams::get($params, $username, 'username', true);
        PostParams::get($params, $fullname, 'fullname', true);
        PostParams::get($params, $recaptchaResponse, 'recaptchaResponse', $_ENV['ENVIRONMENT'] !== 'dev');
        ParamValidator::validateUsername($username);

        // validate recaptcha
        if ($_ENV['ENVIRONMENT'] !== 'dev') {
            Recaptcha::validateResponse($recaptchaResponse);
        } else {
            $this->log->info('DEV environment bypassing recaptcha requirement');
        }

        // create signup
        $signupService = new SignupService($this->db, $this->log, $this->entityManager);
        $signupResponse = $signupService->createSignup($username, $fullname);
        $params['signupId'] = $signupResponse['signupId'];
        $params['usernameValidated'] = $signupResponse['usernameValidated'];

        // create captcha
        $captchaService = new CaptchaService($this->db, $this->log, $this->entityManager);
        $captchaId = $captchaService->initCaptcha($params['signupId']);

        $returnResponse = [
            'signupId' => $signupResponse['signupId'],
            'captchaId' => $captchaId,
            'usernameValidated' => $signupResponse['usernameValidated'],
        ];

        return Response::success($response, $returnResponse, 200);
    }



    public function reinitCaptcha(RequestInterface $request, ResponseInterface $response, array $args)
    {
        // get params and form validations
        $params = (array)$request->getParsedBody();
        $this->log->info("Reinit Captcha POST data: " . print_r($params, true));

        PostParams::get($params, $signupId, 'signupId');
        ParamValidator::validateSignupId($signupId);

        // reinit captcha
        $captchaService = new CaptchaService($this->db, $this->log, $this->entityManager);
        $captchaId = $captchaService->reinitCaptcha($signupId);

        return Response::success($response, array('captchaId' => $captchaId), 200);
    }



    public function sendSms(RequestInterface $request, ResponseInterface $response, array $args)
    {
        //get params and form validations
        $params = (array)$request->getParsedBody();
        $this->log->info("Send SMS Parameters Retrieved: " . print_r($params, true));

        PostParams::get($params, $signupId, 'signupId');
        PostParams::get($params, $captchaId, 'captchaId');
        PostParams::get($params, $smsPhoneNumber, 'smsPhoneNumber');

        ParamValidator::validateSignupId($signupId);
        ParamValidator::validateCaptchaId($captchaId);
        ParamValidator::validatePhoneNumber($smsPhoneNumber);

        // check if signup and captcha records are active
        $signupService = new SignupService($this->db, $this->log, $this->entityManager);
        $signupService->checkSignupRecord($signupId);

        $captchaService = new CaptchaService($this->db, $this->log, $this->entityManager);
        $captchaService->checkCaptchaRecord($captchaId, $signupId);

        // validate sms verification phone number
        $validator = new Validator($this->db, $this->log, $this->entityManager);
        $smsCode = $validator->validatePhoneNumber($signupId, $smsPhoneNumber);

        $sendSmsResponse = [
            'success' => true,
        ];

        if ($_ENV['ENVIRONMENT'] === 'dev') {
            $this->log->info('DEV environment providing SMS pin in response');
            $sendSmsResponse['smsCode'] = $smsCode;
        }

        return Response::success($response, $sendSmsResponse, 200);
    }



    public function validate(RequestInterface $request, ResponseInterface $response, array $args)
    {
        // get params and form validations
        $params = (array)$request->getParsedBody();
        $this->log->info("Signup Parameters Retrieved: " . print_r($params, true));

        PostParams::get($params, $signupId, 'signupId');
        PostParams::get($params, $captchaId, 'captchaId');

        $hasUsername = PostParams::get($params, $username, 'username', false);
        $hasPhoneNumber = PostParams::get($params, $phoneNumber, 'phoneNumber', false);
        $hasSMSCode = PostParams::get($params, $smsCode, 'smsCode', false);
        $hasPostalCode = PostParams::get($params, $postalCode, 'postalCode', false);
        $hasCountry = PostParams::get($params, $country, 'country', false);
        $hasOverrideCode = PostParams::get($params, $overrideCode, 'overrideCode', false);

        // check if at least one parameter to validate is present
        if (!$hasUsername && !$hasPhoneNumber && !$postalCode && !$hasOverrideCode) {
            throw CustomException::noValidateParametersException();
        }

        // possible validations
        ParamValidator::validateSignupId($signupId);
        ParamValidator::validateCaptchaId($captchaId);

        if ($hasUsername)
            ParamValidator::validateUsername($username);

        if ($hasPhoneNumber)
            ParamValidator::validatePhoneNumber($phoneNumber);

        if ($hasSMSCode) {
            ParamValidator::validateSmsCode($smsCode);
        }

        if ($hasPostalCode && $hasCountry) {

            ParamValidator::validatePostalCode($postalCode);
            ParamValidator::validateCountry($country);
        }

        if ($hasOverrideCode) {
            $this->log->info("Validating Override code");
            ParamValidator::validateOverrideCode($overrideCode);
            $responseOutput['overrideCodeValidated'] = true;
        }

        $captchaService = new CaptchaService($this->db, $this->log, $this->entityManager);
        $captchaService->checkCaptchaRecord($captchaId, $signupId);

        $validator = new Validator($this->db, $this->log, $this->entityManager);

        $responseOutput = [];
        if ($hasUsername) {
            $this->log->info("Validating Username: $username");
            $validator->validateUsername($signupId, $username);
            $responseOutput['usernameValidated'] = true;
        }

        if ($hasPhoneNumber) {
            if ($hasSMSCode) {
                $this->log->info("Validating SMS Code");
                $this->db->checkSmsCode($signupId, $phoneNumber, $smsCode);
                $responseOutput['smsCodeValidated'] = true;
            }

            $responseOutput['phoneNumberValidated'] = true;
        }

        if ($hasPostalCode && $hasCountry) {
            $this->log->info("Validating Postal Code: $postalCode, $country");
            $validator->validatePostalCode($signupId, $postalCode, $country);
            $responseOutput['postalCodeValidated'] = true;
        }

        return Response::success($response, $responseOutput, 200);
    }



    public function completeSignup(RequestInterface $request, ResponseInterface $response, array $args)
    {
        // get params and form validations
        $params = (array)$request->getParsedBody();
        $this->log->info("Signup POST data: " . print_r($params, true));

        PostParams::get($params, $signupId, 'signupId');
        PostParams::get($params, $captchaId, 'captchaId');
        PostParams::get($params, $username, 'username');
        PostParams::get($params, $phoneNumber, 'phoneNumber');
        PostParams::get($params, $smsCode, 'smsCode');
        PostParams::get($params, $postalCode, 'postalCode');
        PostParams::get($params, $name, 'name');
        PostParams::get($params, $password, 'password');
        PostParams::get($params, $companyName, 'companyName');
        PostParams::get($params, $companyPhoneNumber, 'companyPhoneNumber');
        PostParams::get($params, $companyWebsite, 'companyWebsite', false);
        PostParams::get($params, $directContactNumber, 'directContactNumber');
        PostParams::get($params, $cardToken, 'cardToken', $_ENV['ENVIRONMENT'] !== 'dev');

        ParamValidator::validateSignupId($signupId);
        ParamValidator::validateCaptchaId($captchaId);
        ParamValidator::validateUsername($username);
        ParamValidator::validatePhoneNumber($phoneNumber);
        ParamValidator::validateSmsCode($smsCode);
        ParamValidator::validatePostalCode($postalCode);
        ParamValidator::validateName($name);
        ParamValidator::validatePassword($password, $username);
        ParamValidator::validateCompanyName($companyName);
        ParamValidator::validatePhoneNumber($companyPhoneNumber, 'companyPhoneNumber');
        ParamValidator::validatePhoneNumber($directContactNumber, 'directContactNumber');
        ParamValidator::validateWebsite($companyWebsite, 'companyWebsite');

        if ($_ENV['ENVIRONMENT'] !== 'dev') {
            ParamValidator::validateCardToken($cardToken);
        }

        $this->log->info("Signup Parameters Validated");

        $captchaService = new CaptchaService($this->db, $this->log, $this->entityManager);
        $captchaService->checkCaptchaRecord($captchaId, $signupId);

        $this->db->checkPhoneNumberIsUnused($phoneNumber);

        $signupService = new SignupService($this->db, $this->log, $this->entityManager);
        $signupService->checkFullSignupRecord($signupId, $username, $phoneNumber, $smsCode);

        $this->log->info("Signup, Captcha, And SMS Phone Number Validated");

        $companyInfo = [
            'name' => $companyName,
            'phoneNumber' => $companyPhoneNumber,
            'address' => $postalCode,
            'website' => $companyWebsite,
            'email' => $username,
        ];

        $userInfo =  [
            'username' => $username,
            'password' => $password,
            'name' => $name,
            'phoneNumber' => $directContactNumber,
        ];

        SkyetelUtil::createOrgAndAccountAndCard($companyInfo, $userInfo, $cardToken, $this->log);

        $this->log->info("Signup Org/Account created");

        $signupService->completeSignup($signupId);

        $email = new Email($this->log);
        $email->sendNewSignupEmail($companyInfo, $userInfo);

        return Response::success($response, array('success' => true), 200);
    }


    
    public function deactivateExpiredSignups(RequestInterface $request, ResponseInterface $response, array $args)
    {
        try {
            $signupService = new SignupService($this->db, $this->log, $this->entityManager);
            $signupService->deactivateExpiredSignups();

            $captchaService = new CaptchaService($this->db, $this->log, $this->entityManager);
            $captchaService->deactivateExpiredCaptchas();

            return Response::success($response, array('success' => true), 200);
        } catch (\Exception $e) {
            $this->log->warning("Deactivating Expired Signups Failed: " . print_r($e, true));
            throw $e;
        }
    }
}
```

---
#### Signup Service
```
<?php

declare(strict_types=1);

namespace App\Application\Services;


use App\Application\Entities\Signup;
use \Doctrine\ORM\EntityManager;
use Psr\Log\LoggerInterface;

use App\Application\RandomStringGenerator;
use App\Application\Database;
use App\Application\CustomException;
use App\Application\Validator;

class SignupService
{
    private $connection;
    public function __construct(private Database $db, private LoggerInterface $log, private EntityManager $entityManager)
    {
        $this->log = $log;
        $this->entityManager = $entityManager;
        $this->db = $db;
        $this->connection = $this->entityManager->getConnection();
    }



    public function getAllSignups()
    {
        $signupRepository = $this->entityManager->getRepository(Signup::class);

        // Doctrine requires a custom query builder here to get an PHP array of the results
        // EntityRepository->findAll() returns an array of objects, not what we want
        $query = $signupRepository->createQueryBuilder('s')->getQuery();
        $signups = $query->getArrayResult();

        if (empty($signups)) {
            throw new \Exception('No signups found');
        }

        return $signups;
    }



    public function createSignup(string $username, string $fullname): array
    {
        // check username availability
        $validator = new Validator($this->db, $this->log, $this->entityManager);
        $validator->validateUsernameWithoutPersistingData($username);

        // generate new signupID
        $idGenerator = new RandomStringGenerator(true, true, $_ENV['ID_LENGTH']);
        $signupId = $idGenerator->generate();

        // init new signup to database by ORM
        $signup = new Signup($signupId, $username, $fullname);
        $this->entityManager->persist($signup);
        $this->entityManager->flush();

        return [
            'signupId' => $signup->getId(),
            'usernameValidated' => true,
        ];
    }



    public function checkSignupRecord(string $signupId)
    {
        $activeHours = $_ENV['SIGNUP_ACTIVE_HOURS'];

        $signupRepository = $this->entityManager->getRepository(Signup::class);
        $signup = $signupRepository->findOneBy(['id' => $signupId]);

        $currentDatetime = new \DateTime();
        if (
            $signup === null ||
            $signup->getCompleted() === 1 ||
            $signup->getActive() === 0 ||
            $signup->getCreatedAt() < $currentDatetime->modify('-' . $activeHours . ' hours')
        ) {
            throw CustomException::invalidSignupIdException();
        }

        $this->log->info('Captcha record checked');
    }



    public function checkFullSignupRecord(string $signupId, string $username, string $phoneNumber, string $smsCode)
    {
        $signupRepository = $this->entityManager->getRepository(Signup::class);
        $signup = $signupRepository->findOneBy(['id' => $signupId, 'username' => $username, 'smsCode' => $smsCode, 'phoneNumber' => $phoneNumber]);

        if ($signup === null) {
            throw CustomException::invalidSignupIdException();
        }

        $this->log->info('Full signup record checked');
    }



    public function completeSignup(string $signupId)
    {
        $signupRepository = $this->entityManager->getRepository(Signup::class);
        $signup = $signupRepository->findOneBy(['id' => $signupId]);

        $signup->setCompleted(1);
        $signup->setActive(0);
        $this->entityManager->flush();

        $this->log->info('Signup completed');
    }


    
    public function deactivateExpiredSignups()
    {
        $this->db->deactivateExpiredSignups();

        $this->log->info('Expired signups deactivated');
    }
}

```

#### Signup Entity
```
<?php

declare(strict_types=1);

namespace App\Application\Entities;

use Doctrine\ORM\Mapping\Entity;
use Doctrine\ORM\Mapping\Table;
use Doctrine\ORM\Mapping\Id;
use Doctrine\ORM\Mapping\Column;

#[Entity]
#[Table('signups')]
class Signup
{
    #[Id]
    #[Column]
    private string $id;

    #[Column(name: 'username')]
    private string $username;

    #[Column(name: 'fullname')]
    private string $fullName;

    #[Column(name: 'phone_number')]
    private string $phoneNumber;

    #[Column(name: 'sms_code')]
    private string $smsCode;

    #[Column(name: 'validated_address')]
    private string $validatedAddress;

    #[Column(options: ['default' => 0])]
    private int $completed;

    #[Column(name: 'create_time', options: ['default' => 'CURRENT_TIMESTAMP'])]
    private \DateTime $createdAt;

    #[Column(options: ['default' => 1])]
    private int $active;

    public function __construct($signupId, $username, $fullname)
    {
        $this->id = $signupId;
        $this->username = $username;
        $this->fullName = $fullname;
        $this->phoneNumber = "not set";
        $this->smsCode = "not set";
        $this->validatedAddress = "not set";
        $this->completed = 0;
        $this->createdAt = new \DateTime("now");
        $this->active = 1;
    }

    // Getters and Setters
    public function getId(): string
    {
        return $this->id;
    }

    public function setId(string $id)
    {
        $this->id = $id;
    }

    public function getUsername(): string
    {
        return $this->username;
    }

    public function setUsername(string $username)
    {
        $this->username = $username;
    }

    public function getFullName(): string
    {
        return $this->fullName;
    }

    public function setFullName(string $fullName)
    {
        $this->fullName = $fullName;
    }

    public function getPhoneNumber(): string
    {
        return $this->phoneNumber;
    }

    public function setPhoneNumber(string $phoneNumber)
    {
        $this->phoneNumber = $phoneNumber;
    }

    public function getSmsCode(): string
    {
        return $this->smsCode;
    }

    public function setSmsCode(string $smsCode)
    {
        $this->smsCode = $smsCode;
    }

    public function getValidatedAddress(): string
    {
        return $this->validatedAddress;
    }

    public function setValidatedAddress(string $validatedAddress)
    {
        $this->validatedAddress = $validatedAddress;
    }

    public function getCompleted(): int
    {
        return $this->completed;
    }

    public function setCompleted(int $completed)
    {
        $this->completed = $completed;
    }

    public function getCreatedAt(): \DateTime
    {
        return $this->createdAt;
    }

    public function setCreatedAt(\DateTime $createdAt)
    {
        $this->createdAt = $createdAt;
    }

    public function getActive(): int
    {
        return $this->active;
    }

    public function setActive(int $active)
    {
        $this->active = $active;
    }
}

```

[Return to main menu](../README.md)