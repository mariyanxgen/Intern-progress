/**
 * @file controller.js
 *
 * Interacts with web-server for sending and recieving data.
 */

/**
 * Global variable to set value of failed record requests
 * to true/false.
 */
var g_record_request_failed = false;

var NOT_ALLOWED_ERROR_CODES = [500];

var SETUP_URL = '/timeline/api/codevideos/setup/';

function call_animate_save(elem_id, flag, animate_save_func) {
    if (animate_save_func) {
        // This is not null in case of react code editor
        animate_save_func(flag);
    }
    else if (typeof animate_save !== "undefined") {
        // There exists global animate_save function
        animate_save(elem_id, flag);
    }
}

/**
 * Removes readonlyRanges key from list of changeset.
 * This is added with code to remove duplciates to offload 
 * extra processing at the server.
 */
function removeReadonlyRanges(changesets) {
    var newChangesets = {};
    
    changesets.forEach(function(element, index){
        var action = element.delta['action'];
        var start = element.delta['start']['row']+element.delta['start']['column']
        var end = element.delta['end']['row']+element.delta['end']['column']
        var lines = element.delta['lines'].join()
        var timestamp = element['timestamp'];
        var prevTimestamp = changesets[index-1] ? changesets[index-1]['timestamp'] : null;
        
        if (timestamp == prevTimestamp) {
            timestamp+=1;
            element['timestamp'] = timestamp
        } 
    
        var dup_id = action+start+end+lines+timestamp;
        newChangesets[dup_id] = {
            initial_state: { code: element.source },
            delta: element.delta,
            timestamp: element.timestamp
        }
    });

    return Object.values(newChangesets);
}

/**
 * Sends changesets to server.
 */
function record_changesets(changesets, finalCode, url, retry, callback, animate_save) {
    if(!(url))
        return;

    var post_data = {
        changesets: changesets ? removeReadonlyRanges(changesets) : [],
        final_code: finalCode,
        last_modified: changesets && changesets.length ? changesets[changesets.length-1].timestamp : new Date().getTime()
    };
    // Show Saving...
    $('.editor-save-code').html('Saving...');

    $.ajax({
        url: url,
        type: 'POST',
        retry: retry,
        post_data: post_data,
        contentType: 'application/json; charset=utf-8',
        async: true,
        changesets: changesets,
        data: JSON.stringify(post_data),
        success: function(response_obj) {
            if(response_obj.status && response_obj.status.toLowerCase() === 'ok') {
                call_animate_save('.editor-save-code', true, animate_save);
                if(g_record_request_failed) {
                    g_record_request_failed = false;
                    if(typeof finish_all_recordings !== 'undefined') {
                        finish_all_recordings();
                    }
                }
                
                // call success callback if defined
                if(callback != undefined && typeof(callback) == "function") {
                    callback();
                }

            } else {
                call_animate_save('.editor-save-code', false, animate_save);
            }
        },
        error: function(err) {
            call_animate_save('.editor-save-code', false, animate_save);

            if(!this.async)
                return;
            
            // Don't retry for NOT_ALLOWED_ERROR_CODES
            if(NOT_ALLOWED_ERROR_CODES.indexOf(err.status)>=0)
                return;

            g_record_request_failed = true;
            this.retry += 1;
            var retry = this.retry;
            var async = this.async;
            var changesets = this.changesets;
            var url = this.url;
            // retry callback
            var retry_func = (function() {
                return function() {
                    record_changesets(changesets, finalCode, url, retry, callback, animate_save);
                };
            })(changesets, url, retry, async);

            setTimeout(retry_func, 10000);
            
        }
    });
};

/**
 * Setup video on server.
 */
function setup_timeline(setupCodeVideoURL, video_obj, callback) {
    $.ajax({
        url: setupCodeVideoURL || SETUP_URL,
        type: 'POST',
        data: video_obj,
        dataType: 'json',
        callback: callback,
        success: function(data, status, xhr) {
            this.callback(xhr.status, data);
        },
        error: function(xhr, err) {
            if (typeof Sentry !== 'undefined'){
                Sentry.withScope(function(scope) {
                    scope.setExtra('origin', 'controller.js - ajax error');
                    scope.setExtra('data', { 'url': setupCodeVideoURL, 'status': xhr.status, 'responseText': xhr.responseText, 'payload': video_obj ? JSON.stringify(video_obj) : null });
                    Sentry.captureException(new Error('ajax error in code player setup'));
                });
            }
            this.callback(xhr.status, xhr.statusText)
        }
    });
};
