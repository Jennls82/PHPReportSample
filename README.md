
<?php
echo "<pre>";

//open csv for writing
//for each $arrEmails...
//make API call to get contact_id
//make API call to get the contact record (so you have first name, etc.)
//populate csv row
//make API call to get activity stream (AS)
//grab that AS, foreach one...
//switch on the type_id, grabbing whatever's needed based on what you find in each varying AS object
//populate csv row
//close csv

//start with an array of email addresses

$arrEmails = array('jsutherland@net-results.com');

$arrRows = array();

$arrHeader = array(
	'contact_id',
	'first_name',
	'last_name',
	'email_address',
	'company',
	'title',
	'work_phone',
	'date',
	'activity',
	'visit'
);

$arrRows[] = $arrHeader;


foreach ($arrEmails as $strEmail) {

	$arrContactRow = array(
		null,
		null,
		null,
		null,
		null,
		null,
		null,
		null,
		null,
		null
	);

	//call for contact id
	$controller = 'Contact';
	$method = 'getContactIdByEmailAddress';
	//$strResponse = execute_json_api_call($controller, $method, $params);
	$params = array('email_address' => $strEmail);

	$strResponse = execute_json_api_call_as_customer($controller, $method, $params, 'z_api_user+vectorworks@net-results.com', 'R3sults!');
	$objResponse = json_decode($strResponse);

	//	print_r($objResponse->result->contact_id);
//		print_r($objResponse);

	//call for contact record
	$controller = 'Contact';
	$method = 'getOne';
	$params = array('contact_id' => $objResponse->result->contact_id);

	$strResponse = execute_json_api_call_as_customer($controller, $method, $params, 'z_api_user+vectorworks@net-results.com', 'R3sults!');
	$objResponse = json_decode($strResponse);

//	print_r($objResponse);
	//csv row here
	$arrContactRow[0] = $objResponse->result->contact_id;
	$arrContactRow[1] = $objResponse->result->first_name;
	$arrContactRow[2] = $objResponse->result->last_name;
	$arrContactRow[3] = $objResponse->result->email_address;
	$arrContactRow[4] = $objResponse->result->company;
	$arrContactRow[5] = $objResponse->result->title;
	$arrContactRow[6] = $objResponse->result->work_phone;

	$arrRows[] = $arrContactRow;

	//call for contact AS
	$controller = 'ContactActivityHistory';
	$method = 'getContactActivity';
	$params = array(
			'contact_id' => $objResponse->result->contact_id,
			'offset' => 0,
			'limit' => 25,
			'order_by' => 'contact_activity_history_date',
			'order_dir' => 'DESC'
		);

	$strResponse = execute_json_api_call_as_customer($controller, $method, $params, 'z_api_user+vectorworks@net-results.com', 'R3sults!');
	//$strResponse = execute_json_api_call($controller, $method, $params);
	$objResponse = json_decode($strResponse);

		//echo "!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!AS activity:<br />";
		//print_r($objResponse);

	foreach($objResponse->result->results as $objActivityHistory) {

		$arrHistoryRow = array(
			null,
			null,
			null,
			null,
			null,
			null,
			null,
			null,
			null,
			null
		);

		$arrPageViewRow = null;


//		$arrHistoryRow[6] = $objActivityHistory->page_views;


		$objActivityDate = new DateTime($objActivityHistory->contact_activity_history_date);
		$objActivityDate->setTimezone(new DateTimeZone('America/Denver'));
		$strActivityDate = $objActivityDate->format('n/j/Y');
		$arrHistoryRow[7] = $strActivityDate;

		switch ($objActivityHistory->contact_activity_history_type_id) {

			case 1:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Contact created by User</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 2:
				$arrHistoryRow[8] = 'Contact created by import of: ' . $objActivityHistory->contact_import_file;
				break;

			case 3:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Contact created by SalesForce</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 4:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Contact created by Visit</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 5:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Email Bounce</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;
//WHAT LINK DID THEY CLICK:
			case 6:
				$arrHistoryRow[8] = 'Email Click: ' . $objActivityHistory->email_name;

				if(isset($objActivityHistory->email_name)) {
				$arrHistoryRow[9] = 'Link: ' . $objActivityHistory->click_href;
			}

				break;

			case 7:
				$arrHistoryRow[8] = 'Email Open: ' . $objActivityHistory->email_name;
				break;

			case 8:
				$arrHistoryRow[8] = 'Email Send: ' . $objActivityHistory->email_name;
				break;

			case 9:
				$arrHistoryRow[8] = 'Email Unsubscribe' . $objActivityHistory->email_name;
				break;

			//no type 10!

			case 11:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Uploaded to SalesForce</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 12:
				//call for visit object
				$controller = 'Visit';
				$method = 'getOne';
				$params = array('visit_id' => $objActivityHistory->visit_id);

				$strResponse = execute_json_api_call_as_customer($controller, $method, $params, 'z_api_user+vectorworks@net-results.com', 'R3sults!');
				$objResponse = json_decode($strResponse);
				//echo "<h1>Visit::getOne</h1><br />";
				//print_r($objResponse);
//FORMAT THIS DURATION:
				$intDuration = $objResponse->result->total_duration/60;

				$arrHistoryRow[8] = "Website Visit: {$objResponse->result->total_pages} Pages Viewed, {$intDuration} Minute Duration";

				foreach ($objResponse->result->visits as $objVisit){
					$arrPageViewRows = array();

					//echo "<b>*********HERE COMES OUR VISIT OBJECT... ******** </b><br />";
					//print_r($objVisit);

					foreach ($objVisit->accesses as $objAccess) {
						//echo "<b>********* BEER! COMES OUR ACCESS OBJECT ... ******** </b><br />";
						//print_r($objAccess);

						$arrPageViewRow = array(
							null,
							null,
							null,
							null,
							null,
							null,
							null,
							null,
							null,
							null
						);

						$arrPageViewRow[9] = 'Page View: ' . $objAccess->full_page;
//FORMAT THIS DATE/TIME:
						$objActivityDate = new DateTime($objAccess->time);
						$objActivityDate->setTimezone(new DateTimeZone('America/Denver'));
						$strActivityDate = $objActivityDate->format('n/j/Y h:i:sa');
						$arrPageViewRow[10] = 'Page View Timestamp: ' . $strActivityDate;

						if (isset($objAccess->duration)) {
							$strDuration = $objAccess->duration;
						} else {
							$strDuration = 'N/A';
						}
						$arrPageViewRow[11] = 'Page View Duration: ' . $strDuration;
						$arrPageViewRows[] = $arrPageViewRow;
					}
				}
				break;

			case 13:
				$srtStatus = '';
				$srtStatus = $objActivityHistory->email_list_status == 'Added' ? 'added to: ' : 'removed from: ';
				$arrHistoryRow[8] = 'List Membership ' . $srtStatus . $objActivityHistory->email_list_name;
				break;

			case 14:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Imported from data.com</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 15:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Lead Ownership Changed</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 16:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Contact created from Sugar Lead</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 17:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Contact created from Sugar Contact</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 18:
				$arrHistoryRow[8] = 'Uploaded to Sugar: ' . $objActivityHistory->description;
				break;

			case 19:
				$arrHistoryRow[8] = 'Lead Score Adjustment: ' . $objActivityHistory->contact_score_adjustment . ' by ' . $objActivityHistory->contact_score_adjustment_adjuster;
				break;

			case 20:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Web Form Submission</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 21:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Contact Subscription Add</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 22:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Contact Subscription Remove</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			//no type 23!

			case 24:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Uploaded SalesForce Task</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 25:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Contact created by SalesLogix</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 26:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Uploaded to SalesLogix</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 27:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Uploaded SalesLogix Task</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 28:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Receive Conversation</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 29:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Open Conversation</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 30:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Click Conversation</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 31:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Conversation Visit</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 32:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Conversation Bounce</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 33:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Conversation Unsubscribe</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 34:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Contact created by GoToWebinar</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 35:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Registered for Webinar</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 36:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Attended Webinar</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 37:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>MA Form Submission</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 38:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Content Download</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 39:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Lead Stage</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 40:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>MA Form View</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 41:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Contact created from Dynamics Contact</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 42:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Contact created from Dynamics Lead</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 43:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Uploaded to Dynamics</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 44:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Uploaded Dynamics Activity</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			case 45:
				$arrHistoryRow[8] = $objActivityHistory->contact_activity_history_type;
				echo "<b>Uploaded Sugar Task</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			//no type 46!
			//no type 47!

			case 48:
				$arrHistoryRow[8] = 'Video viewed: ' . $objActivityHistory->contact_activity_history_description;
				echo "<b>Video</b> " . $objActivityHistory->contact_activity_history_date . $objActivityHistory->contact_activity_history_type . '<br />';
				break;

			default:
				echo "history_type_id = " . $objActivityHistory->contact_activity_history_type_id . '<br />';
		}
		$arrRows[] = $arrHistoryRow;
		if (count($arrPageViewRows) > 0) {
			foreach($arrPageViewRows as $arrPageViewRow) {
				$arrRows[] = $arrPageViewRow;
			}
		}
		unset($arrPageViewRows);
	}
};

echo "<b>ARRROWS</b><br />";
print_r($arrRows);

$fh = fopen('/tmp/file.csv', 'w');
foreach ($arrRows as $arrRow) {
	fputcsv($fh, $arrRow);
}
fclose($fh);


